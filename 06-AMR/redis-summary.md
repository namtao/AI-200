# Tổng hợp Azure Managed Redis — AI-200

> Khoá: AI-200 · Redis — 3 modules: Data Operations · Event Messaging · Vector Storage

---

## Bức tranh tổng thể 3 modules

```
Module 1: DATA OPERATIONS          Module 2: EVENT MESSAGING          Module 3: VECTOR STORAGE
─────────────────────────          ─────────────────────────          ──────────────────────────
Redis as cache/session store   →   Redis as message broker        →   Redis as vector database
                                                                        
Cache-aside pattern                Pub/Sub: broadcast             RediSearch + embeddings
TTL & expiration                   Streams: reliable task queue   FLAT vs HNSW algorithms
String/Hash/List/Numeric           Hybrid: cả hai                 COSINE/L2 distance metrics
```

**Redis trong AI stack:** Sub-millisecond latency cho session data + event-driven coordination giữa AI services + semantic search trên embeddings — tất cả trong một managed service.

---

## Module 1 — Implement Data Operations

### Cốt lõi

**Azure Managed Redis** = Redis Enterprise, managed bởi Microsoft. Không phải community Redis.

**3 Caching Strategies:**

| Pattern | Cơ chế | Use case |
|---|---|---|
| Cache-aside (Data Cache) | App check Redis → HIT return · MISS query DB → store → return | User profiles, model results |
| Content Cache | Pre-store static content | Headers, navigation, banners |
| Session Store | Session ID trong cookie, full data trong Redis | Auth tokens, conversation context |

**4 Tiers:**

| Tier | Memory:vCPU | Đặc điểm |
|---|---|---|
| Memory Optimized | 8:1 | Dev/test, memory-intensive |
| Balanced | 4:1 | Standard production |
| Compute Optimized | 2:1 | Max throughput |
| Flash Optimized | NVMe disk | Large datasets, cost-effective |

**4 Data Types:**

| Type | Commands | Dùng khi |
|---|---|---|
| String | SET, GET, MSET, MGET | Text, serialized data |
| Hash | HSET, HGET, HGETALL | Structured objects |
| List | LPUSH, RPUSH, LPOP, RPOP, LRANGE | Queue, feed |
| Numeric | INCR, DECR, INCRBY | Counters, rate limiting |

**TTL/Expiration:**

```python
r.setex('session:abc', 60, 'data')     # Set + expire atomic — dùng cái này
r.expire('key', 3600)                   # Add expiration cho existing key
ttl = r.ttl('key')                      # -1 = exists no TTL · -2 = not exist · N = N giây
r.persist('key')                        # Remove expiration
```

---

## Module 2 — Implement Event Messaging

### Pub/Sub — Broadcast Pattern

**Đặc điểm cốt lõi:**
- At-most-once delivery — offline subscriber = miss messages vĩnh viễn
- No persistence — messages không lưu, chỉ tồn tại khi distribute
- Fire-and-forget — publisher không biết ai nhận được

**Dùng khi:** Cache invalidation tất cả instances, config updates tất cả services, real-time notifications.

**Không dùng khi:** Task queues, guaranteed delivery, work distribution, cần replay.

```python
# Publisher
redis_client.publish('ai:models:updated', 'embeddings-v2')

# Subscriber
pubsub = redis_client.pubsub()
pubsub.subscribe('ai:models:updated')
for message in pubsub.listen():
    if message['type'] == 'message':          # 'pmessage' với PSUBSCRIBE
        handle(message['channel'], message['data'])
```

### Streams — Reliable Task Queue

**Đặc điểm cốt lõi:**
- Persistence — messages không mất khi worker offline
- Consumer groups — mỗi task chỉ xử lý bởi một worker
- Acknowledgment — XACK mark complete, không XACK = pending → retry
- Phải trim thủ công — không tự expire

**Dùng khi:** AI inference requests, document processing, pipelines cần exactly-once.

```python
# Add task
redis.xadd('ai:queue', {'user': '123', 'prompt': '...'}, maxlen=10000)

# Worker process
redis.xgroup_create('ai:queue', 'workers', id='0', mkstream=True)
messages = redis.xreadgroup('workers', f'worker-{id}', {'ai:queue': '>'}, count=5, block=5000)
for stream, tasks in messages:
    for task_id, data in tasks:
        process(data)
        redis.xack('ai:queue', 'workers', task_id)    # BẮTBUỘC

# Retry stuck tasks
pending = redis.xpending('ai:queue', 'workers', '-', '+', 10)
for task in pending:
    if task['time_since_delivered'] > 300000:    # 5 phút
        redis.xclaim('ai:queue', 'workers', f'worker-{id}', 0, [task['message_id']])
```

### Decision: Pub/Sub vs Streams

| Scenario | Pattern |
|---|---|
| Cache invalidation → tất cả API instances clear | Pub/Sub |
| Inference request → một worker xử lý | Streams |
| Config update → tất cả services reload | Pub/Sub |
| Document upload → pipeline step | Streams |
| Real-time dashboard (nhiều clients) | Pub/Sub |
| Queue AI jobs với auto-retry | Streams |
| Complex system (queue + notify) | **Hybrid** |

---

## Module 3 — Implement Vector Storage

### RediSearch Vector Index

```python
from redis.commands.search.field import TextField, VectorField
from redis.commands.search.indexDefinition import IndexDefinition, IndexType

schema = (
    TextField("title"),
    VectorField("embedding", "HNSW", {
        "TYPE": "FLOAT32",           # Standard cho AI
        "DIM": 1536,                  # Phải match model output chính xác
        "DISTANCE_METRIC": "COSINE"  # COSINE text · L2 image
    })
)

redis_client.ft("idx:docs").create_index(
    fields=schema,
    definition=IndexDefinition(prefix=["doc:"], index_type=IndexType.HASH)
)
```

### FLAT vs HNSW

| | FLAT | HNSW |
|---|---|---|
| Accuracy | 100% exact | 95-99% approximate |
| Speed | O(n) linear | Sub-10ms even 1M+ vectors |
| Memory overhead | Thấp | Cao hơn (graph) |
| Dùng khi | <10K, dev/test | >10K, production |

### Distance Metrics

| Metric | Đo gì | Dùng khi |
|---|---|---|
| COSINE | Angle (ignores magnitude) | Text embeddings (OpenAI, Cohere) |
| L2 | Straight-line distance | Image embeddings, spatial |
| IP | Dot product | Pre-normalized embeddings only |

### Vector Types

| Type | Bytes/dim | 1536-dim | Dùng khi |
|---|---|---|---|
| FLOAT32 | 4 | 6 KB | Standard — mọi AI application |
| FLOAT64 | 8 | 12 KB | Research — không dùng cho AI |

### Hash vs JSON cho Vector Storage

| | Hash | JSON |
|---|---|---|
| Vector encoding | `embedding.tobytes()` | `embedding.tolist()` |
| IndexType | `IndexType.HASH` | `IndexType.JSON` |
| Field syntax | `"fieldname"` | `"$.fieldname"` + `as_name` |
| Nested objects | ❌ | ✅ |
| Performance | Nhanh hơn | Slightly slower |
| Chọn khi | Flat models, max perf | Nested data, flexibility |

### KNN Query

```python
import numpy as np

# Ingest
redis_client.hset("doc:001", mapping={
    "title": "Azure AI",
    "embedding": np.array(embed, dtype=np.float32).tobytes()    # tobytes() cho Hash
})

# Query
query_bytes = np.array(query_embed, dtype=np.float32).tobytes()
query = (
    Query("*=>[KNN 5 @embedding $query_vec AS score]")
    .return_fields("title", "score")
    .sort_by("score")
    .dialect(2)    # BẮT BUỘC cho vector queries
)
results = redis_client.ft("idx:docs").search(query, query_params={"query_vec": query_bytes})
# Score COSINE: 0.0 = identical · 2.0 = opposite — thấp hơn = similar hơn
```

---

## CLI / Code Quick Reference

### Connection

```python
# Password (dev)
r = redis.Redis(host='instance', port=10000, ssl=True, decode_responses=True, password='key')

# Entra ID (production)
from azure.identity import DefaultAzureCredential
from redis_entraid.cred_provider import create_from_default_azure_credential
cred_provider = create_from_default_azure_credential(("https://redis.azure.com/.default",))
r = redis.Redis(host='instance', port=10000, ssl=True, decode_responses=True,
                credential_provider=cred_provider)
```

### Data Operations

```python
r.set('key', 'value')                          # String set
r.get('key')                                    # String get
r.mset({'k1': 'v1', 'k2': 'v2'})              # Multi set
r.mget('k1', 'k2')                             # Multi get → list

r.hset('user:1', mapping={'name': 'Alice'})    # Hash set
r.hget('user:1', 'name')                       # Hash get field
r.hgetall('user:1')                            # Hash get all → dict

r.incr('counter')                              # Atomic increment
r.exists('k1', 'k2', 'k3')                    # Returns COUNT (0-3), không phải bool

r.setex('session', 60, 'data')                 # Set + expire atomic
r.ttl('key')                                   # -1 no TTL · -2 not exist · N = N giây

pipe = r.pipeline()
pipe.hgetall('user:1')
pipe.hgetall('user:2')
results = pipe.execute()                       # Batch trong 1 round trip
```

### Pub/Sub

```python
# Publish
redis_client.publish('channel', 'message')

# Subscribe exact
pubsub = redis_client.pubsub()
pubsub.subscribe('ch1', 'ch2')
for msg in pubsub.listen():
    if msg['type'] == 'message':               # 'pmessage' với psubscribe
        handle(msg['channel'], msg['data'])

# Pattern subscribe
pubsub.psubscribe('ai:*')
```

### Streams

```python
# Add (với trim)
redis.xadd('queue', {'field': 'value'}, maxlen=10000, approximate=True)

# Setup group (idempotent)
try: redis.xgroup_create('queue', 'workers', id='0', mkstream=True)
except: pass

# Worker read
msgs = redis.xreadgroup('workers', f'worker-{id}', {'queue': '>'}, count=10, block=5000)

# Acknowledge
redis.xack('queue', 'workers', task_id)

# Check pending
pending = redis.xpending('queue', 'workers', '-', '+', 10)

# Trim periodic
redis_client.xtrim('queue', maxlen=10000, approximate=True)
```

### Vector

```python
# Create index
from redis.commands.search.field import VectorField, TextField
from redis.commands.search.indexDefinition import IndexDefinition, IndexType

schema = (TextField("title"), VectorField("embedding", "HNSW",
    {"TYPE": "FLOAT32", "DIM": 1536, "DISTANCE_METRIC": "COSINE"}))
redis_client.ft("idx").create_index(schema,
    definition=IndexDefinition(prefix=["doc:"], index_type=IndexType.HASH))

# KNN search
from redis.commands.search.query import Query
q = Query("*=>[KNN 5 @embedding $vec AS score]").return_fields("title","score").sort_by("score").dialect(2)
results = redis_client.ft("idx").search(q, query_params={"vec": embed.tobytes()})

# Hybrid (filter + KNN)
q = Query("@category:{docs}=>[KNN 3 @embedding $vec AS score]").dialect(2)

# Range query
q = Query("@embedding:[VECTOR_RANGE 0.2 $vec]=>{$YIELD_DISTANCE_AS: score}").dialect(2)
```

---

## Exam Traps (≥8)

### 1. Port — 3 services, 3 ports

| Service | Port |
|---|---|
| Azure Managed Redis | **10000** |
| Azure Cache for Redis | 6380 |
| Community Redis | 6379 |

Câu hỏi thường dùng "default TLS port" → answer 10000.

### 2. KEYS vs SCAN trong production

`KEYS pattern` = block toàn bộ Redis server. Với production database hàng triệu keys, một lệnh KEYS có thể block server hàng giây. **Luôn dùng SCAN** với cursor-based pagination.

### 3. EXISTS trả về số lượng, không phải boolean

```python
r.exists('k1', 'k2', 'k3')    # Trả về 0, 1, 2, hoặc 3 — không phải True/False
```

Sai khi viết: `if r.exists('k1', 'k2'):` với mong đợi boolean check.

### 4. TTL -1 vs -2

- **-1** = key TỒN TẠI, không có expiration
- **-2** = key KHÔNG TỒN TẠI

Hai giá trị âm dễ nhầm. -1 không có nghĩa là "expired 1 giây trước".

### 5. `setex()` vs `expire()` — atomic vs 2-step

- `setex(key, seconds, value)` = set + expire trong 1 atomic operation
- `expire(key, seconds)` = add expiration cho key **đã tồn tại** — cần 2 operations → race condition potential

### 6. Pub/Sub với multiple workers = duplicate processing

3 workers subscribe cùng channel → 3 copies của message → 3x GPU compute wasted. Dùng Streams consumer groups khi cần exactly-one-worker.

### 7. Streams phải trim thủ công

Tasks không tự expire trong Streams. Quên `maxlen` → OOM. Phải dùng:
```python
redis.xadd('queue', data, maxlen=10000, approximate=True)
# hoặc
redis_client.xtrim('queue', maxlen=10000, approximate=True)
```

### 8. `pmessage` vs `message` type

- `SUBSCRIBE` → `message['type'] == 'message'`
- `PSUBSCRIBE` → `message['type'] == 'pmessage'`

Sai type = bỏ qua tất cả messages mà không có error.

### 9. `.dialect(2)` bắt buộc cho vector queries

Quên `.dialect(2)` → syntax error hoặc wrong results. Không phải optional.

### 10. DIM phải match chính xác embedding model output

Model output 1536 dimensions, index define 1024 → ingestion fail hoặc query error. Mismatch không có warning tường minh.

### 11. tobytes() vs tolist() — Hash vs JSON encoding

- Hash: `embedding.tobytes()` → binary bytes
- JSON: `embedding.tolist()` → list of floats

Nhầm → index tạo thành công nhưng vector search fail.

### 12. EF_RUNTIME ≠ max results

- `KNN 10` = trả về 10 results
- `EF_RUNTIME 200` = explore 200 graph nodes để tìm candidates

Hai parameter hoàn toàn khác nhau.

---

## Assessment Summaries

### Module 1 Assessment — 4/4

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Default TLS port | **10000** |
| 2 | Command không dùng trong production | **KEYS** (dùng SCAN) |
| 3 | Set key + expiration atomically | **`setex()`** |
| 4 | TTL = -1 nghĩa là gì | **Key exists, no expiration set** |

### Module 2 Assessment — 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Khác biệt pub/sub vs Streams | **Pub/sub active subscribers only, Streams persist** |
| 2 | Reliable pipeline với auto retry | **Streams consumer groups** |
| 3 | Command thêm message vào Stream | **XADD** |
| 4 | Best use case cho pub/sub | **Broadcast real-time status updates** |
| 5 | Coordinate multiple workers, no duplicate | **XREADGROUP với consumer group** |

### Module 3 Assessment — 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Distance metric cho text embeddings | **COSINE** |
| 2 | Chọn HNSW khi nào | **>10K vectors, fast queries, 95-99% accuracy OK** |
| 3 | Data type cho AI embeddings | **FLOAT32** |
| 4 | Hash vs JSON | **Hash = flat models, max efficiency** |
| 5 | EF_RUNTIME parameter | **Speed/accuracy tradeoff — graph nodes examined** |

---

## Redis vs Cosmos DB vs PostgreSQL cho AI Workloads

| Tiêu chí | Azure Managed Redis | Cosmos DB | PostgreSQL (pgvector) |
|---|---|---|---|
| **Latency** | Sub-millisecond | Low ms (single-digit) | Low-medium ms |
| **Primary strength** | Speed, caching, messaging | Global scale, multi-model | SQL queries, relational data |
| **Vector search** | RediSearch (FLAT/HNSW) | Built-in DiskANN | pgvector (IVFFlat/HNSW) |
| **Data persistence** | In-memory (optional AOF/RDB) | Fully persistent | Fully persistent |
| **Data model** | Key-value, Hash, Stream | Document, Graph, Table | Relational tables |
| **Session/Cache** | ✅ Native, sub-ms | ❌ Overkill | ❌ Overkill |
| **Event streaming** | ✅ Pub/Sub + Streams | ❌ | ❌ |
| **Conversation history** | ✅ Fast read/write | ✅ Long-term storage | ✅ Long-term + queries |
| **RAG vector store** | ✅ Low-latency retrieval | ✅ Scalable | ✅ Rich SQL filtering |
| **Cost profile** | Highest (all in-memory) | High (global replicas) | Lower (disk-based) |
| **Scale** | Cluster, Active-Active | Global multi-region | Vertical + read replicas |

### Khi nào chọn cái nào cho AI:

**Chọn Redis khi:**
- Cần sub-millisecond latency (session data, conversation context, cached embeddings)
- Event-driven AI pipeline coordination (task queues, cache invalidation broadcast)
- Hot path data: user profiles, rate limiting, real-time recommendations

**Chọn Cosmos DB khi:**
- Global distribution required (multi-region active-active writes)
- Multi-model access (same data via SQL, Mongo, Gremlin API)
- Large-scale document storage với vector search (DiskANN)

**Chọn PostgreSQL/pgvector khi:**
- Đã có relational data, cần thêm vector search
- Complex SQL filtering kết hợp với semantic search
- Cost-sensitive workloads không cần sub-ms latency
- Rich metadata filtering trên nhiều dimensions

**Pattern phổ biến — dùng kết hợp:**
```
Request → Redis (session + cache check)
           ↓ Cache MISS
         PostgreSQL / Cosmos DB (persistent storage + vector search)
           ↓ Result
         Redis (cache result với TTL)
           ↓
         Redis Streams (queue async processing)
           ↓
         Redis Pub/Sub (broadcast completion notification)
```

---

*Tổng hợp hoàn thành — 3 modules, 14 files (9 bài học + 3 assessments + 1 module-assessment tổng hợp + file này)*
