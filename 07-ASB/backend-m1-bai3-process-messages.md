# Bài 3 — Process Messages Reliably

> Khoá: AI-200 · Service Bus — Queue and process AI operations with Azure Service Bus

---

## Receive Modes

| Mode | Behavior | AI workload |
|---|---|---|
| **Peek-lock** (default) | Lock message, chờ consumer settle | **Recommended — at-least-once delivery** |
| Receive-and-delete | Remove ngay khi deliver | Mất message nếu crash — không dùng cho critical AI |

---

## Settlement Operations (Peek-lock)

| Method | Effect | Dùng khi |
|---|---|---|
| `complete_message()` | Remove permanently | Processing thành công |
| `abandon_message()` | Release lock, requeue ngay | Transient error (timeout, network) |
| `dead_letter_message()` | Move to DLQ | Permanent error (malformed JSON, unsupported model) |
| `defer_message()` | Keep in queue, remove from delivery | Cần xử lý sau, access bằng sequence number |

---

## Peek-lock Processing Loop

```python
import json
from azure.servicebus import ServiceBusClient, ServiceBusReceiveMode
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

with ServiceBusClient(
    fully_qualified_namespace="<namespace>.servicebus.windows.net",
    credential=credential
) as client:
    with client.get_queue_receiver(
        queue_name="inference-requests",
        receive_mode=ServiceBusReceiveMode.PEEK_LOCK,
        max_wait_time=30
    ) as receiver:
        for msg in receiver:
            try:
                payload = json.loads(str(msg))
                result = run_inference(payload)
                save_result(result, payload.get("document_id"))
                receiver.complete_message(msg)                    # Success

            except json.JSONDecodeError:
                receiver.dead_letter_message(
                    msg,
                    reason="MalformedPayload",
                    error_description="Message body is not valid JSON"
                )                                                  # Permanent error → DLQ

            except ModelNotFoundError:
                receiver.dead_letter_message(
                    msg,
                    reason="UnsupportedModel",
                    error_description=f"Model {payload.get('model')} is not available"
                )                                                  # Permanent error → DLQ

            except TransientServiceError:
                receiver.abandon_message(msg)                     # Transient → retry
```

---

## Max Delivery Count và Dead-Letter Queue

**Max delivery count** (default = 10): Khi message bị abandon quá nhiều lần → tự động move to DLQ với reason `MaxDeliveryCountExceeded`.

Tùy chỉnh:
- Thấp (3-5): Move nhanh hơn đến DLQ, free capacity cho healthy messages
- Cao: Nhiều retry cho transient errors

**Dead-Letter Queue (DLQ):**
- Subqueue tự động attached vào mọi queue/subscription
- Persist messages cho đến khi explicitly receive + complete
- Mỗi message có `dead_letter_reason` và `dead_letter_error_description`

### Đọc DLQ

```python
from azure.servicebus import ServiceBusSubQueue

with client.get_queue_receiver(
    queue_name="inference-requests",
    sub_queue=ServiceBusSubQueue.DEAD_LETTER,   # Không tự build path
    max_wait_time=10
) as dlq_receiver:
    for msg in dlq_receiver:
        print(f"Reason: {msg.dead_letter_reason}")
        print(f"Error: {msg.dead_letter_error_description}")
        print(f"Correlation ID: {msg.correlation_id}")
        print(f"Delivery count: {msg.delivery_count}")
        dlq_receiver.complete_message(msg)
```

Dùng `ServiceBusSubQueue.DEAD_LETTER` enum, không construct path thủ công.

### Reprocess DLQ

```python
with client.get_queue_receiver("inference-requests",
                               sub_queue=ServiceBusSubQueue.DEAD_LETTER) as dlq_receiver:
    with client.get_queue_sender("inference-requests") as sender:
        for msg in dlq_receiver:
            new_message = ServiceBusMessage(
                body=msg.body,
                content_type=msg.content_type,
                correlation_id=msg.correlation_id,
                application_properties=msg.application_properties
            )
            sender.send_messages(new_message)
            dlq_receiver.complete_message(msg)
```

Confirm root cause fixed trước khi resubmit — nếu chưa fix, messages cycle lại DLQ.

---

## Long-running AI Operations — AutoLockRenewer

Queue lock max = **5 phút** (default = 1 phút). Long inference có thể exceed → Service Bus make available for another consumer → duplicate processing.

```python
from azure.servicebus import AutoLockRenewer

renewer = AutoLockRenewer()

with client.get_queue_receiver("inference-requests", max_wait_time=30) as receiver:
    for msg in receiver.receive_messages():
        renewer.register(receiver, msg, max_lock_renewal_duration=600)  # Renew up to 10 phút
        result = run_long_inference(msg)
        receiver.complete_message(msg)

renewer.close()
```

**Alternative cho rất long operations:** Two-phase approach — nhận message, record vào tracking store, complete message nhanh → separate process pick up từ tracking store.

---

## Bản chất bài này là gì?

**Một câu:** Reliable message processing = Peek-lock + 4 settlement operations + DLQ handling + AutoLockRenewer cho long-running AI tasks — không để message bị mất hay stuck.

### So sánh với AWS SQS / RabbitMQ / Kafka

| | AWS SQS | RabbitMQ | Kafka | Service Bus |
|---|---|---|---|---|
| "Peek-lock" equivalent | Visibility timeout | Ack/Nack | Manual offset commit | Peek-lock |
| Settlement options | Delete hoặc visibility reset | Ack / Nack (requeue hoặc dead-letter) | Offset commit | complete / abandon / dead_letter / defer |
| Dead-letter | Separate DLQ config | `x-dead-letter-exchange` | Không native | Automatic subqueue, `ServiceBusSubQueue.DEAD_LETTER` |
| Max delivery count | MaxReceiveCount (redrive policy) | `x-death` header count | Không native | Default 10, configurable |
| Long-running lock renewal | Visibility extension API | heartbeat | N/A | `AutoLockRenewer` |
| Defer (remove from delivery) | Không | Không | Không | `defer_message()` + sequence number |

**`defer_message()` là gì:** Message không bị xóa, không bị DLQ — bị "ẩn" khỏi normal delivery. Chỉ retrieve được bằng sequence number. Dùng khi cần xử lý sau theo điều kiện khác.

**Exam trap:** Lock max = 5 phút, KHÔNG phải 1 phút (1 phút là default setting có thể config). Nếu AI inference > 5 phút mà không dùng AutoLockRenewer → duplicate processing. Đây là câu hỏi phổ biến.

---

## Checklist ghi nhớ cho AI-200

- [ ] **Peek-lock** = at-least-once, recommended cho AI · Receive-and-delete = có thể mất message
- [ ] `complete_message()` = success · `abandon_message()` = transient retry · `dead_letter_message()` = permanent error
- [ ] `defer_message()` = remove from delivery, access bằng sequence number
- [ ] Max delivery count default = **10** → exceed = auto move to DLQ với `MaxDeliveryCountExceeded`
- [ ] DLQ properties: `dead_letter_reason` · `dead_letter_error_description` · `delivery_count`
- [ ] Đọc DLQ: `sub_queue=ServiceBusSubQueue.DEAD_LETTER` (không tự build path)
- [ ] Monitor DLQ = detect systemic failures (model deployment issue, malformed requests)
- [ ] Lock max duration: **5 phút** · `AutoLockRenewer` auto-renew trong background
- [ ] Reprocess: confirm fix before resubmit

---

*Module Assessment tiếp theo*

---

[← Bài 2](./backend-m1-bai2-message-structure.md) · [🏠 Mục lục](../README.md) · [Assessment →](./backend-m1-module-assessment.md)
