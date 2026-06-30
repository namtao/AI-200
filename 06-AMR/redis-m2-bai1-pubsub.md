# Bài 1 — Publish and Subscribe to Events with Redis Pub/Sub

> Khoá: AI-200 · Redis — Implement event messaging with Azure Managed Redis

---

## Pub/Sub Pattern là gì?

Publishers gửi messages đến **named channels** — không cần biết ai đang listen. Subscribers listen một hoặc nhiều channels và nhận messages ngay khi được published. **Loose coupling, event-driven architecture.**

**Use case AI:** Khi new customer message arrive → publish đến "conversations:new" → Sentiment analyzer, Intent classifier, Context manager đều subscribe và bắt đầu parallel processing ngay lập tức.

---

## Đặc điểm quan trọng của Pub/Sub

| Đặc điểm | Ý nghĩa |
|---|---|
| **At-most-once delivery** | Chỉ deliver đến subscribers đang kết nối. Subscriber offline → miss messages |
| **No message persistence** | Messages không được lưu — chỉ tồn tại lúc đang distribute |
| **Fire-and-forget** | Publisher không biết ai nhận được |
| **High throughput** | Xử lý high volume, low latency |

---

## Channel Naming

```
ai:conversations:123       # Conversation-specific events
ml:model:predictions       # ML prediction results
ai:training:status         # Model training status
embeddings:cache:refresh   # Embedding cache updates
```

---

## Publish

```python
# Publish cache invalidation event
redis_client.publish('ai:models:updated', f'{model_name}:{version}')

# Publish embedding refresh
redis_client.publish('ai:embeddings:refresh', collection_id)
```

---

## Subscribe (cụ thể channels)

```python
pubsub = redis_client.pubsub()
pubsub.subscribe('ai:models:updated', 'ai:embeddings:refresh')

for message in pubsub.listen():
    if message['type'] == 'message':
        channel = message['channel']
        data = message['data']

        if channel == 'ai:models:updated':
            clear_model_cache(data)
        elif channel == 'ai:embeddings:refresh':
            refresh_embeddings(data)
```

---

## Pattern Subscribe (nhiều channels cùng lúc)

```python
pubsub = redis_client.pubsub()
pubsub.psubscribe('ai:*')    # Tất cả channels bắt đầu bằng 'ai:'

for message in pubsub.listen():
    if message['type'] == 'pmessage':
        pattern = message['pattern']  # 'ai:*'
        channel = message['channel']  # Tên channel thực tế
        data = message['data']
        handle_ai_event(channel, data)
```

---

## WebSocket + Pub/Sub

```python
from fastapi import WebSocket
import redis.asyncio as redis

async def redis_listener(websocket: WebSocket):
    r = redis.Redis(host='your-redis', port=10000, ssl=True,
                   password='key', decode_responses=True)

    async with r.pubsub() as pubsub:
        await pubsub.subscribe('ai:predictions:ready')

        async for message in pubsub.listen():
            if message['type'] == 'message':
                await websocket.send_json({
                    'type': 'prediction',
                    'data': message['data']
                })
```

---

## Khi nào KHÔNG dùng Pub/Sub

- ❌ Processing tasks chỉ nên chạy một lần (pub/sub = tất cả subscribers đều nhận = duplicate processing)
- ❌ Load balancing across workers
- ❌ Cần delivery guarantees hay acknowledgments
- ❌ Cần message persistence hay replay

---

## Bản chất bài này là gì?

**Một câu:** Pub/Sub là broadcast pattern — mọi subscriber active đều nhận cùng message, không có persistence, dùng cho notifications và cache invalidation, không dùng cho task queues.

### So sánh với Redis Streams

| | Pub/Sub | Streams |
|---|---|---|
| Delivery | At-most-once | At-least-once (với XACK) |
| Persistence | ❌ | ✅ |
| Offline subscriber | Miss messages | Nhận khi reconnect |
| Delivery target | Tất cả subscribers | Một consumer per group |
| Phù hợp | Broadcast, notifications | Task queues, pipelines |

**Fire-and-forget là đặc điểm không thể thay đổi:** Publisher không biết có ai nhận. Nếu cần biết message đã được xử lý, Pub/Sub là sai tool.

**Exam trap — pmessage vs message:** `PSUBSCRIBE` pattern → `message['type'] == 'pmessage'`, `SUBSCRIBE` channel → `message['type'] == 'message'`. Sai type = bỏ qua toàn bộ messages.

---

## Checklist ghi nhớ cho AI-200

- [ ] Pub/sub = **broadcast** — tất cả active subscribers đều nhận
- [ ] **At-most-once** — offline subscriber = miss messages
- [ ] **No persistence** — messages không được lưu
- [ ] `PUBLISH channel message` · `SUBSCRIBE channel` · `PSUBSCRIBE pattern`
- [ ] `message['type'] == 'message'` (SUBSCRIBE) · `'pmessage'` (PSUBSCRIBE)
- [ ] Dùng pub/sub: cache invalidation, config updates, broadcast notifications
- [ ] KHÔNG dùng: task queues, guaranteed delivery, work distribution

---

[← Assessment](./redis-m1-module-assessment.md) · [🏠 Mục lục](../README.md) · [Bài 2 →](./redis-m2-bai2-streams.md)
