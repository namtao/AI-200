# Bài 3 — Choose Between Broadcast and Coordinated Distribution

> Khoá: AI-200 · Redis — Implement event messaging with Azure Managed Redis

---

## Vấn đề khi dùng sai pattern

3 workers subscribe cùng pub/sub channel:
- Pub/sub → cả 3 worker nhận cùng message → **3x duplicate processing**, GPU cycles lãng phí, inconsistent results

3 workers dùng Streams consumer group:
- Mỗi task → đúng 1 worker → **automatic load balancing**, không duplicate

---

## Pub/Sub — Broadcast (tất cả đều nhận)

**Behavior:** Mỗi message deliver đến **tất cả** active subscribers.

```python
# Publisher
redis_client.publish('ai:cache:invalidate', 'embeddings-v2')

# Subscriber 1 (API server 1) — nhận
# Subscriber 2 (API server 2) — nhận
# Subscriber 3 (Background worker) — nhận
# Tất cả đều nhận cùng message
```

**Dùng pub/sub khi:**
- Cache invalidation — tất cả API instances phải clear cache cùng lúc
- Config update — tất cả services cần reload config mới
- Real-time monitoring — nhiều systems xử lý cùng metrics theo cách khác nhau
- Event notification heterogeneous services — analytics, billing, recommendation đều cần biết

---

## Streams Consumer Groups — Coordinated (mỗi task một worker)

**Behavior:** Mỗi message trong group → chỉ **một** consumer nhận.

```python
# Worker 1 nhận messages 1, 3, 5
# Worker 2 nhận messages 2, 4, 6
# Worker 3 nhận messages 7, 8, 9
# Automatic distribution — không cần code load balancing

messages = redis_client.xreadgroup(
    groupname='workers',       # Cùng group name
    consumername=f'worker-{worker_id}',  # Unique per worker
    streams={'ai:inference:queue': '>'},
    count=10
)
```

**Dùng Streams khi:**
- Task chỉ xử lý một lần (inference request, document processing)
- Load balancing — thêm worker = automatic workload sharing
- Cần guaranteed delivery và acknowledgments
- Cần retry khi worker crash

---

## Decision Table

| Scenario | Pattern | Lý do |
|---|---|---|
| Model updated → clear cache tất cả API instances | **Pub/Sub** | Tất cả cần clear cùng lúc |
| Inference request → xử lý bởi một worker | **Streams** | Chỉ một worker xử lý |
| Config change → tất cả services reload | **Pub/Sub** | Broadcast đến tất cả |
| Document upload → một pipeline step | **Streams** | Exactly-once processing |
| Real-time dashboard updates | **Pub/Sub** | Nhiều clients nhận cùng data |
| Queue AI jobs, auto-retry on failure | **Streams** | Reliability cần thiết |

---

## Hybrid Pattern

Nhiều AI architecture cần cả hai:

```python
@app.post("/api/process-document")
async def process_document(doc: UploadFile):
    # Streams: Queue actual processing work (coordinated)
    task_id = redis_client.xadd('ai:document:queue', {
        'doc_id': doc.filename,
        'user': current_user.id
    })

    # Pub/Sub: Broadcast notification cho real-time updates
    redis_client.publish('ai:documents:events', json.dumps({
        'event': 'document_received',
        'task_id': task_id,
        'user': current_user.id
    }))

    return {"task_id": task_id}

# Workers: xử lý từ Stream (coordinated — mỗi doc 1 worker)
def document_processor_worker():
    # ... consumer group code

# Services: listen event từ pub/sub (broadcast — WebSocket, analytics, monitoring)
async def event_listener_service():
    async with redis_client.pubsub() as pubsub:
        await pubsub.subscribe('ai:documents:events')
        async for message in pubsub.listen():
            handle_event(message['data'])
```

---

## Summary

| | Pub/Sub | Streams |
|---|---|---|
| **Message delivery** | All active subscribers | One consumer per group |
| **Persistence** | ❌ | ✅ |
| **Retry on failure** | ❌ | ✅ |
| **Scaling pattern** | Broadcast to all | Distribute across workers |
| **Use for** | Notifications, cache invalidation | Task queues, AI pipelines |

---

## Bản chất bài này là gì?

**Một câu:** Pub/Sub broadcast cho tất cả cần biết (cache invalidation, config updates), Streams consumer groups cho exactly-one-worker processing (inference tasks, document pipelines) — hybrid architecture dùng cả hai.

### So sánh Pub/Sub vs Streams Consumer Groups

| | Pub/Sub | Streams Consumer Groups |
|---|---|---|
| 3 workers nhận 1 message | 3 copies (duplicate) | 1 worker nhận (distributed) |
| Cache invalidation | ✅ Mọi instance cần clear | ❌ Chỉ một instance clear |
| AI inference task | ❌ 3x GPU waste | ✅ Load balanced |
| Thêm worker | Tự động nhận broadcast | Tự động nhận workload share |
| Worker offline | Miss messages | Tasks vẫn trong queue |

**Duplicate processing là bug, không phải feature:** 3 workers subscribe Pub/Sub channel nhận inference task = 3 GPU instances chạy cùng computation. Với AI workloads đắt tiền, đây là lãng phí nghiêm trọng.

**Exam trap — hybrid là đáp án đúng cho complex systems:** Nhiều câu hỏi cho scenario phức tạp. Streams cho actual work + Pub/Sub cho notifications là pattern chuẩn, không phải chọn một trong hai.

---

## Checklist ghi nhớ cho AI-200

- [ ] Pub/sub = **broadcast** — 3 subscribers = 3 copies của message
- [ ] Streams consumer group = **exactly one** consumer per message
- [ ] Cache invalidation → **pub/sub** (tất cả instances cần clear)
- [ ] Inference task queue → **Streams** (mỗi task một worker)
- [ ] Pub/sub: dùng khi tất cả cần phản ứng với cùng event
- [ ] Streams: dùng khi chỉ một worker xử lý task
- [ ] **Hybrid:** Streams cho actual work + pub/sub cho notifications
- [ ] Thêm worker vào consumer group → automatic workload distribution

---

*Module Assessment tiếp theo*
