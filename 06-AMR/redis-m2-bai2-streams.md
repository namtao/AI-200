# Bài 2 — Implement Task Queues with Redis Streams

> Khoá: AI-200 · Redis — Implement event messaging with Azure Managed Redis

---

## Streams là gì?

**Append-only task queue** với persistence, consumer groups, và acknowledgment. Khác pub/sub: messages không mất khi subscriber offline, mỗi task chỉ được xử lý bởi một worker, failed tasks có thể retry.

**Use case:** Document processing API → XADD task vào Stream → return ngay → background workers process tại pace của mình → nếu crash, task vẫn còn trong pending list.

---

## Lợi ích so với Pub/Sub

| | Pub/Sub | Streams |
|---|---|---|
| Message persistence | ❌ | ✅ |
| Delivery guarantee | ❌ | ✅ (với XACK) |
| Auto retry on crash | ❌ | ✅ (pending list) |
| Work distribution | ❌ Duplicate | ✅ Consumer groups |
| Processing history | ❌ | ✅ |

---

## Core Commands

### XADD — Thêm task vào queue

```python
stream_id = redis.xadd('ai:inference:requests', {
    'user_id': '12345',
    'model': 'gpt-4',
    'prompt': 'Summarize this document...',
    'priority': 'high'
})
# Returns ID: '1699980000000-0' (auto-generated, ordered)
```

### XREADGROUP — Worker lấy task

```python
messages = redis.xreadgroup(
    groupname='workers',
    consumername='worker-instance-1',
    streams={'ai:inference:requests': '>'},    # '>' = new messages only
    count=10,
    block=5000    # Wait 5s nếu queue trống
)
```

### XACK — Mark task complete

```python
redis.xack('ai:inference:requests', 'workers', task_id)
```

---

## Setup Consumer Group

```python
# Tạo một lần (idempotent)
try:
    redis.xgroup_create('ai:inference:queue', 'workers', id='0', mkstream=True)
except redis.ResponseError:
    pass  # Group already exists — OK
```

- `id='0'` = start from beginning
- `mkstream=True` = tạo stream nếu chưa tồn tại

---

## Worker Processing Loop

```python
def process_ai_tasks():
    # Setup (idempotent)
    try:
        redis.xgroup_create('ai:inference:queue', 'workers', id='0', mkstream=True)
    except:
        pass

    while True:
        messages = redis.xreadgroup(
            groupname='workers',
            consumername=f'worker-{os.getpid()}',
            streams={'ai:inference:queue': '>'},
            count=5,
            block=5000
        )

        for stream, tasks in messages:
            for task_id, task_data in tasks:
                try:
                    result = run_inference(task_data['prompt'], task_data['model'])
                    save_result(task_id, result)
                    redis.xack('ai:inference:queue', 'workers', task_id)    # Mark done
                except Exception as e:
                    log_error(f"Task {task_id} failed: {e}")
                    # Task stays in pending — sẽ retry
```

---

## Handle Failed Tasks (XPENDING + XCLAIM)

```python
def retry_stuck_tasks():
    pending = redis.xpending('ai:inference:queue', 'workers', '-', '+', 10)

    for task in pending:
        task_id = task['message_id']
        idle_time = task['time_since_delivered']    # milliseconds

        if idle_time > 300000:    # 5 phút = likely crashed
            redis.xclaim('ai:inference:queue', 'workers',
                        f'worker-{os.getpid()}', 0, [task_id])
            # Now reprocess task_id
```

---

## Memory Management — Trimming

Tasks **không tự expire** trong Streams → phải trim:

```python
# Trim khi XADD (keep last 10,000)
redis.xadd('ai:inference:queue', task_data, maxlen=10000, approximate=True)

# Hoặc trim định kỳ
redis_client.xtrim('ai:documents:queue', maxlen=10000, approximate=True)
```

---

## Monitor Queue Health

```python
def get_queue_stats():
    info = redis_client.xinfo_stream('ai:documents:queue')
    groups = redis_client.xinfo_groups('ai:documents:queue')

    return {
        'total_tasks': info['length'],
        'pending_tasks': groups[0]['pending'],
        'consumers': groups[0]['consumers']
    }
```

---

## Trade-offs của Streams

- **Memory management là trách nhiệm của bạn** — phải trim
- Latency cao hơn pub/sub một chút (persistence overhead) — OK cho AI workloads
- Nhiều code hơn pub/sub (groups, ACK, pending checks)
- Tasks không auto-expire

---

## Bản chất bài này là gì?

**Một câu:** Redis Streams là append-only task queue với consumer groups — mỗi task chỉ được xử lý bởi một worker, failed tasks ở lại pending list để retry, nhưng phải trim thủ công.

### So sánh với Redis List (LPUSH/RPOP)

| | Redis List | Redis Streams |
|---|---|---|
| Consumer groups | ❌ | ✅ |
| Acknowledgment | ❌ | ✅ (XACK) |
| Retry on crash | ❌ (task lost) | ✅ (pending list) |
| Message history | ❌ | ✅ |
| Auto load balancing | ❌ | ✅ |
| Phù hợp | Simple queue | AI pipelines, reliability required |

**XACK là bắt buộc, không phải optional:** Không XACK → task mãi ở pending list → XPENDING tăng vô tận → phải XCLAIM thủ công. Nếu worker crash sau khi xử lý xong nhưng chưa XACK, task sẽ được retry.

**Exam trap — phải trim thủ công:** Streams không tự expire. Quên `maxlen` hoặc `XTRIM` → Redis OOM. Đây là khác biệt quan trọng so với queue với TTL tự động.

---

## Checklist ghi nhớ cho AI-200

- [ ] `XADD stream fields` → add task, returns auto-generated ordered ID
- [ ] `XREADGROUP group consumer {stream: '>'} count block` → get new tasks
- [ ] `'>'` symbol = chỉ lấy messages chưa deliver đến bất kỳ consumer nào
- [ ] `XACK stream group task_id` → mark task complete
- [ ] `XGROUP CREATE stream group '0' MKSTREAM` → setup (idempotent)
- [ ] Tasks không được ACK → ở lại **pending list** → có thể retry
- [ ] `XPENDING` → find stuck tasks · `XCLAIM` → reassign
- [ ] **Trim** bắt buộc: `maxlen=10000` hoặc `XTRIM`
- [ ] Consumer groups → automatic work distribution across workers

---

*Bài tiếp theo: Bài 3 — Choose between broadcast and coordinated distribution*
