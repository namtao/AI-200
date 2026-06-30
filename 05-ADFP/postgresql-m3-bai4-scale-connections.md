# Bài 4 — Scale for High-Volume Workloads + Connection Optimization

> Khoá: AI-200 · PostgreSQL — Optimize vector search in Azure Database for PostgreSQL

---

## Vertical Scaling

| Tier | Memory/vCore | Dùng khi |
|---|---|---|
| Burstable | 2 GB | Dev, low traffic |
| General Purpose | 4 GB | Balanced production |
| **Memory Optimized** | **8 GB** | **Large working sets, AI/vector** |

Memory Optimized: giữ HNSW indexes trong memory → ít disk I/O → lower latency.

**Scale up (larger server):** Single-query latency cao, memory pressure, CPU > 70% sustained.
**Scale out (replicas):** Total query volume vượt một server, read-heavy workload.

---

## Read Replicas

- Up to **5 read replicas** per primary
- Asynchronous replication — lag vài milliseconds đến seconds
- Vector similarity searches = ideal candidates (read-only, slight staleness OK)

```sql
-- Check replica lag (run trên replica)
SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS lag_seconds;
```

Khi không cần strong consistency: route vector queries đến replicas.
Khi cần immediate consistency: route đến primary.

---

## Caching Strategy

Tốt cho cache: popular product embeddings, precomputed similar items, stable data.
Không tốt: arbitrary vector queries (infinite query space), rapidly changing data.

```python
import redis, json

redis_client = redis.Redis(host='cache.redis.cache.windows.net', port=6380, ssl=True, password='key')

def get_product_embedding(product_id):
    cached = redis_client.get(f"embedding:{product_id}")
    if cached:
        return json.loads(cached)
    embedding = fetch_from_db(product_id)
    redis_client.setex(f"embedding:{product_id}", 3600, json.dumps(embedding))  # TTL 1 hour
    return embedding
```

---

## Connection Overhead

New connection = 50-200ms: TCP handshake + TLS + authentication + server process allocation.

**Connection limits:**

| Tier | Max connections |
|---|---|
| Burstable 1ms | 50 |
| GP 2 vCores | 859 |
| **GP 4 vCores** | **1,718** |
| MO 4 vCores | 3,437 |

---

## PgBouncer

```bash
az postgres flexible-server parameter set \
    --resource-group myRG --server-name myserver \
    --name pgbouncer.enabled --value true
```

Connect trên port **6432** (không phải 5432).

**3 Pooling Modes:**

| Mode | Connection held | Trade-off |
|---|---|---|
| Session | Entire session | All features, minimal saving |
| **Transaction** | During transaction only | **Best cho vector search** |
| Statement | Per statement | Max saving, no multi-statement txn |

**Transaction mode limitations:**
- `SET` không persist → dùng `SET LOCAL` trong transaction
- Prepared statements không reliable
- `LISTEN/NOTIFY` không support

→ Vector search (simple selects, no session state) không bị ảnh hưởng.

> PgBouncer chỉ có **General Purpose** và **Memory Optimized**. Burstable không hỗ trợ.

---

## Application-level Connection Pool

```python
from psycopg_pool import ConnectionPool

pool = ConnectionPool(
    conninfo="postgresql://user:pass@server.postgres.database.azure.com:6432/db",
    min_size=5,
    max_size=20,
    max_idle=300,       # Close idle after 5 min
    max_lifetime=3600   # Recycle after 1 hour
)

def search(query_embedding, limit=10):
    with pool.connection() as conn:
        with conn.cursor() as cur:
            cur.execute(
                "SELECT id, name, embedding <=> %s AS distance "
                "FROM products ORDER BY embedding <=> %s LIMIT %s",
                (query_embedding, query_embedding, limit)
            )
            return cur.fetchall()
```

**Pool sizing:** `max_size = server_max_connections / app_instances - headroom`

Recycle connections mỗi 30-60 phút (max_lifetime) để maintain health.

---

## Session Parameters cho Vector Queries

```sql
-- Trong transaction — reverts khi transaction kết thúc
SET LOCAL work_mem = '256MB';
SET LOCAL hnsw.ef_search = 200;    -- High recall cho critical queries
SET LOCAL hnsw.ef_search = 20;     -- Fast cho approximate OK
```

`SET LOCAL` an toàn với pooled connections vì reverts sau transaction.

---

## Monitor Capacity

Azure Monitor metrics cần theo dõi:
- CPU % — sustained >70% → scale up
- Memory % — approaching limits → scale up
- Storage IO % — cao → data không trong cache → tăng shared_buffers
- Active connections — approaching limit → enable PgBouncer

---

## Bản chất bài này là gì?

**Một câu:** Scaling Azure Database for PostgreSQL cho vector workload là vertical first (Memory Optimized cho index-in-memory), PgBouncer transaction mode cho connection multiplexing, read replicas cho query volume — thứ tự này quan trọng vì mỗi bước giải quyết vấn đề khác nhau.

### So sánh với Azure Cosmos DB for NoSQL vs Redis Cache vs Amazon Aurora PostgreSQL

| Scaling dimension | Azure DB for PostgreSQL | Cosmos DB for NoSQL | Azure Cache for Redis | Aurora PostgreSQL |
|---|---|---|---|---|
| Vertical scale (compute) | Tier upgrade (restart) | RU/s adjustment (no restart) | Cache size (no restart) | Instance type change |
| Read replicas | 5 replicas, async | Global read replicas | ❌ (no SQL) | 15 replicas |
| Connection pooling | PgBouncer built-in (GP/MO) | N/A (HTTP API) | N/A | RDS Proxy ($) |
| Max connections (4 vCore GP) | 1,718 | No connection concept | 10,000 | 5,000 |
| Caching layer | In-memory (shared_buffers) | Integrated cache | External (Redis) | Buffer pool |
| Query latency target | P95 <50ms (Memory Opt) | <10ms (key-value) | <1ms | P95 <50ms |
| Vector search | pgvector native | ❌ | ❌ | pgvector |

**Read replicas giải quyết throughput, không phải latency — đây là exam trap quan trọng nhất trong bài:** Nếu P95 latency là 150ms và target là 50ms, adding read replica vẫn cho P95 150ms per query. Latency là single-query performance → cần vertical scale (Memory Optimized tier, more vCores, indexes in memory). Throughput là total queries/second → read replicas. Nhầm lẫn hai khái niệm này là lỗi phổ biến nhất trong exam questions về scaling.

**PgBouncer transaction mode làm vô hiệu hóa `SET` parameters — gotcha khi optimize per-query:** `SET hnsw.ef_search = 200` trong transaction mode không persist sau transaction vì connection trả về pool. Phải dùng `SET LOCAL hnsw.ef_search = 200` trong mỗi transaction. Nếu dùng session mode PgBouncer, `SET` persist — nhưng session mode không giúp nhiều cho connection multiplication. Đây là không obvious khi chuyển từ direct connections sang PgBouncer.

---

## Checklist ghi nhớ cho AI-200

- [ ] **Memory Optimized** tier: 8 GB/vCore, best cho vector workloads
- [ ] Scale up → single-query latency · Scale out (replicas) → total throughput
- [ ] Read replicas: up to 5, async, vector search = ideal candidate
- [ ] PgBouncer: port **6432**, **transaction mode** recommended
- [ ] PgBouncer: chỉ GP và MO, không có Burstable
- [ ] Transaction mode: `SET LOCAL` (không phải `SET`)
- [ ] `ConnectionPool(min_size, max_size, max_lifetime=3600)`
- [ ] Pool max_size = server_limit / app_instances - headroom
- [ ] `SET LOCAL hnsw.ef_search` để tune per-transaction accuracy

---

*Module Assessment tiếp theo*

---

[← Bài 3](./postgresql-m3-bai3-data-layout.md) · [🏠 Mục lục](../README.md) · [Assessment →](./postgresql-m3-module-assessment.md)
