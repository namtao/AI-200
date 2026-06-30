# Bài 1 — Choose Between Queues and Topics with Subscriptions

> Khoá: AI-200 · Service Bus — Queue and process AI operations with Azure Service Bus

---

## Service Bus là gì?

Azure Service Bus là managed message broker giúp **decouple** các components trong AI application. Producer gửi message mà không cần biết consumer nào sẽ xử lý — loose coupling, async processing.

---

## Queue — Single-consumer Processing

Mỗi message được deliver đến **đúng một** consumer (competing consumers). Scale horizontally bằng cách thêm workers — Service Bus tự phân phối, không cần code thêm.

```python
from azure.servicebus import ServiceBusClient, ServiceBusMessage
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

with ServiceBusClient(
    fully_qualified_namespace="<namespace>.servicebus.windows.net",
    credential=credential
) as client:
    with client.get_queue_sender("inference-requests") as sender:
        sender.send_messages(ServiceBusMessage("message body"))
```

**Receive modes:**

| Mode | Behavior | Dùng khi |
|---|---|---|
| **Peek-lock** (default) | Lock message, consumer phải settle (complete/abandon/DLQ) | **AI workloads — at-least-once delivery** |
| Receive-and-delete | Remove ngay khi deliver | Throughput ưu tiên, mất message OK |

Peek-lock recommended cho AI: nếu worker crash, lock expire → message available cho worker khác.

**Sessions:** FIFO processing cho related messages (vd. multi-step pipeline: extract → classify → embed).

---

## Topic + Subscriptions — Fan-out

Message được **copy** đến tất cả subscriptions. Mỗi subscription = independent virtual queue với consumer riêng.

```python
# Send to topic
with client.get_topic_sender("inference-results") as sender:
    sender.send_messages(ServiceBusMessage("result body"))

# Receive from subscription
with client.get_subscription_receiver(
    topic_name="inference-results",
    subscription_name="notifications"
) as receiver:
    for msg in receiver:
        print(str(msg))
        receiver.complete_message(msg)
```

**Use case:** Document analysis completes → notification sub (alert user) + audit sub (log compliance) + metrics sub (update dashboard) — tất cả nhận cùng message, process độc lập.

**Lợi ích:** Add subscription mới mà không sửa producer code.

---

## Subscription Filter Rules

Route messages selectively tại broker level — không cần filtering logic trong consumer.

```python
from azure.servicebus.management import ServiceBusAdministrationClient, SqlRuleFilter

admin_client = ServiceBusAdministrationClient(
    fully_qualified_namespace="<namespace>.servicebus.windows.net",
    credential=credential
)

admin_client.create_rule(
    topic_name="inference-results",
    subscription_name="high-priority",
    rule_name="filter-high-priority",
    filter=SqlRuleFilter("priority = 'high'")
)
```

**3 filter types:**
- **Boolean:** TrueFilter (tất cả) hoặc FalseFilter (không ai)
- **SQL filter:** `priority = 'high'`, `model_name = 'gpt-4o'`
- **Correlation filter:** Exact-match, hiệu quả hơn SQL

---

## Queue vs Topic Decision

**Dùng Queue khi:**
- Single processing pipeline xử lý mỗi request
- Cần competing consumers cho horizontal scaling
- Không service nào khác cần cùng message

**Dùng Topic + Subscriptions khi:**
- Multiple independent services cần react cùng message
- Cần content-based routing (filter theo model, priority, type)
- Muốn add consumer mới mà không sửa producer

---

## Bản chất bài này là gì?

**Một câu:** Service Bus là message broker giúp decouple AI components — Queue cho single-consumer processing, Topic+Subscriptions cho fan-out đến nhiều consumers độc lập.

### So sánh với RabbitMQ / AWS SQS / Redis Streams

| | RabbitMQ | AWS SQS | Redis Streams | Azure Service Bus |
|---|---|---|---|---|
| Queue model | Exchange → Queue | Queue only | Consumer groups | Queue + Topic/Subscription |
| Fan-out | Exchange với bindings | SNS → SQS | Multiple consumer groups | Topic với subscriptions |
| Filter | Routing key | Không built-in | Không built-in | SQL filter, Correlation filter |
| At-least-once | Ack/Nack | Visibility timeout | ACK | Peek-lock + settle |
| Sessions (FIFO) | Không native | FIFO queue (separate) | Không | Sessions |
| Managed | Self-hosted | Fully managed | Self-hosted/Redis Cloud | Fully managed |

**Điểm khác biệt then chốt:** Service Bus là broker duy nhất có built-in SQL filter tại subscription level — routing logic ở broker, không phải consumer code.

**Exam trap:** Queue = competing consumers (chỉ MỘT nhận), Topic = fan-out (TẤT CẢ subscriptions nhận copy). Nhầm hai cái này = sai câu hỏi assessment.

---

## Checklist ghi nhớ cho AI-200

- [ ] Queue: mỗi message → **một** consumer · Topic: message → **tất cả** subscriptions (copy)
- [ ] **Peek-lock** = at-least-once · Receive-and-delete = highest throughput, có thể mất message
- [ ] Peek-lock: consumer phải settle (complete/abandon/dead-letter) trong lock duration
- [ ] Lock expire = message available lại → không mất message
- [ ] Sessions = FIFO, single-receiver per session group
- [ ] Subscription filter: SQL filter (`priority = 'high'`) · Correlation filter (exact match)
- [ ] `get_queue_sender()` · `get_topic_sender()` · `get_subscription_receiver()`
- [ ] Add subscription = không cần sửa producer code

---

[← Mục lục](../README.md) · [🏠 Mục lục](../README.md) · [Bài 2 →](./backend-m1-bai2-message-structure.md)
