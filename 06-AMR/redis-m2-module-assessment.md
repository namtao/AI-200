# Module Assessment — Implement Event Messaging with Azure Managed Redis

> Khoá: AI-200 · Redis — Implement event messaging with Azure Managed Redis

---

## Câu 1

**What is the key difference between Redis pub/sub and Redis Streams for event messaging?**

- Pub/sub delivers messages to active subscribers only, while Streams persist messages for consumer groups to process at their own pace ✅
- Pub/sub can handle more messages per second than Streams
- Streams are only for string data while pub/sub works with any data type

**Đáp án: Pub/sub chỉ deliver đến active subscribers, Streams persist**

Đây là sự khác biệt cốt lõi:

**Pub/sub:** At-most-once delivery. Subscriber offline = miss messages forever. Messages không được lưu.

**Streams:** Messages persist trong append-only log. Consumers có thể đọc tại pace của mình. Consumer offline → messages vẫn còn khi reconnect.

Tại sao các đáp án còn lại sai:
- **Pub/sub handle nhiều messages/second hơn:** Không có basis cho claim này. Streams với consumer groups also designed cho high throughput. Performance phụ thuộc vào tier và config, không phải inherently limited.
- **Streams chỉ cho string data:** Sai — cả hai đều work với key-value pairs. Streams fields là strings nhưng không limited so với pub/sub.

---

## Câu 2

**When building a processing pipeline with multiple services that need automatic retry on failure, which pattern should you use?**

- Redis Streams with consumer groups for reliable task coordination and built-in retry handling ✅
- Redis pub/sub for real-time distribution of work items
- A simple Redis List with LPUSH and RPOP

**Đáp án: Redis Streams with consumer groups**

Streams consumer groups cung cấp chính xác những gì cần:

```python
# Task vào pending list khi được delivered
messages = redis.xreadgroup('workers', 'worker-1', {'queue': '>'}, count=5)

for task_id, data in messages:
    try:
        process(data)
        redis.xack('queue', 'workers', task_id)    # Mark complete
    except:
        pass    # Task stays in pending → auto retry
```

Khi worker crash: task ở lại pending list → `XPENDING` detect → `XCLAIM` reassign.

Tại sao các đáp án còn lại sai:
- **Pub/sub:** Fire-and-forget, no persistence, no acknowledgments. Worker offline = task lost. Hoàn toàn không có retry mechanism. Sai pattern cho reliability requirement.
- **Redis List với LPUSH/RPOP:** Simple queue nhưng không có acknowledgments hay consumer groups. Nếu worker crash sau RPOP (item đã removed from list) nhưng trước khi process xong → task lost. Không có automatic retry.

---

## Câu 3

**Which Redis command is used to add a new message to a Stream?**

- XADD ✅
- LPUSH
- PUBLISH

**Đáp án: XADD**

```python
stream_id = redis.xadd('ai:inference:requests', {
    'user_id': '12345',
    'model': 'gpt-4',
    'prompt': 'Summarize this...'
})
# Returns auto-generated ordered ID: '1699980000000-0'
```

`XADD` = add to Stream. Stream commands đều có prefix `X`.

Tại sao các đáp án còn lại sai:
- **LPUSH:** Add to **List** (not Stream). List là simpler data structure, không có consumer groups hay acknowledgments.
- **PUBLISH:** Add message to pub/sub **channel** (not Stream). Pub/sub không persistent.

---

## Câu 4

**Which scenario is best suited for Redis pub/sub messaging?**

- Broadcasting real-time status updates to multiple connected clients or services that are currently listening ✅
- Storing messages that arrive when subscribers are offline and delivering them when subscribers reconnect
- Implementing a reliable work queue where tasks must be processed exactly once

**Đáp án: Broadcasting real-time status updates**

Pub/sub là designed cho broadcast: một message → tất cả active subscribers nhận cùng lúc. Real-time status updates (model training progress, prediction ready, system metrics) phù hợp vì:
- Nhiều services/clients cần biết
- Delivery đến active subscribers là đủ
- Real-time = không cần persistence

Tại sao các đáp án còn lại sai:
- **Storing messages for offline subscribers:** Đây chính xác là điều pub/sub KHÔNG làm. Messages lost khi subscriber offline. Cần Streams cho requirement này.
- **Reliable work queue, exactly-once processing:** Pub/sub deliver đến TẤT CẢ subscribers — không "exactly once" per task. Không có acknowledgments. Không reliable. Cần Streams consumer groups.

---

## Câu 5

**How would you coordinate multiple AI workers processing documents from a Stream to ensure no document is processed by more than one worker simultaneously?**

- Use XREADGROUP to pull tasks from a consumer group, which automatically assigns unacknowledged tasks to different workers ✅
- Use XREAD to have each worker independently read from the Stream without coordination
- Use pub/sub channels with one channel per worker

**Đáp án: XREADGROUP với consumer group**

```python
# Tất cả workers dùng cùng group name → automatic distribution
messages = redis_client.xreadgroup(
    groupname='workers',              # Same group → coordinated
    consumername=f'worker-{worker_id}',  # Unique per worker
    streams={'ai:document:queue': '>'},
    count=5,
    block=5000
)
# Worker 1 nhận task A, Worker 2 nhận task B, không bao giờ duplicate
```

Consumer group tracks: message nào đã delivered cho consumer nào → `>` chỉ return messages chưa deliver đến bất kỳ consumer nào trong group.

Tại sao các đáp án còn lại sai:
- **XREAD (không có group):** Mỗi worker reads từ Stream independently. Nếu 3 workers đọc offset 0, tất cả thấy cùng messages → duplicate processing. XREAD không có coordination mechanism giữa workers.
- **Pub/sub, một channel per worker:** Phức tạp hóa không cần thiết, không scalable (cần biết số workers trước), không automatic load balancing. Producer phải route messages manually đến từng channel — defeats the purpose of dynamic scaling.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Khác biệt pub/sub vs Streams | Pub/sub active subscribers only, Streams persist |
| 2 | Reliable pipeline với auto retry | Streams consumer groups |
| 3 | Command thêm message vào Stream | XADD |
| 4 | Best use case cho pub/sub | Broadcast real-time status updates |
| 5 | Coordinate multiple workers, no duplicate | XREADGROUP với consumer group |

---

*Module hoàn thành.*
