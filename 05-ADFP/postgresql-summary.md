# PostgreSQL — Tổng hợp AI-200

> Dense reference. 3 modules: Azure DB for PostgreSQL · pgvector · Performance Optimization.

---

## Bức tranh tổng thể

```
Module 1: Nền tảng
  ├── Bài 1: Service (tiers, backup, PgBouncer, auth, TLS)
  ├── Bài 2: Schema (types, constraints, FK, indexes, DDL)
  ├── Bài 3: Query (JSONB, keyset pagination, CTE, upsert)
  └── Bài 4: SDK (psycopg v3, pool, retry, COPY)

Module 2: pgvector
  ├── Bài 1: Basics (types, operators, dimensions)
  ├── Bài 2: Indexes (IVFFlat vs HNSW, operator class)
  ├── Bài 3: Lifecycle (rebuild, stale flag, migration, hybrid search)
  └── Bài 4: RAG (2-table design, chunking, context window, citation)

Module 3: Performance
  ├── Bài 1: Tuning (memory, planner, pgvector params)
  ├── Bài 2: Index selection (decision framework, build times)
  ├── Bài 3: Data layout (structured vs JSONB, pre/post-filter, partitioning)
  └── Bài 4: Scale (vertical, replicas, PgBouncer, pool sizing)
```

---

## Module 1: Azure Database for PostgreSQL

### Compute Tiers

| Tier | VM | vCore:Memory | Phù hợp |
|---|---|---|---|
| **Burstable** | B-series | 2 GB/vCore | Dev, test, intermittent load |
| **General Purpose** | D-series | 4 GB/vCore | Production, web app, backend |
| **Memory Optimized** | E-series | 8 GB/vCore | AI, large in-memory, analytics |

**PgBouncer chỉ có General Purpose và Memory Optimized — không có Burstable.**

### Managed Capabilities

| Feature | Detail |
|---|---|
| Backup retention | Default **7 ngày**, max **35 ngày** |
| Backup encryption | AES-256 |
| Point-in-time restore | Tạo **server instance MỚI**, không restore in-place |
| PgBouncer port | **6432** (thay vì 5432) |
| Direct connect port | **5432** |

### Authentication

```python
# Entra auth — token scope
credential.get_token("https://ossrdbms-aad.database.windows.net/.default")

# DefaultAzureCredential: Azure → Managed Identity | Local → Azure CLI
```

### TLS Modes

| Mode | Production? |
|---|---|
| `disable` | Azure reject |
| `require` | Encrypt, no cert validate |
| **`verify-full`** | **Production: encrypt + CA + hostname** |

### Key Data Types

| Type | Dùng khi |
|---|---|
| `BIGSERIAL` | High-volume insert, auto-increment 64-bit |
| `UUID DEFAULT gen_random_uuid()` | Generate ID ngoài DB, distributed sources |
| **`JSONB`** | Flexible/varying structure, indexable via GIN |
| **`TIMESTAMPTZ`** | Temporal data (luôn dùng, PostgreSQL lưu UTC) |
| `TEXT` | Unlimited string, conversation content |
| `CHECK (col IN (...))` | Enforce allowed values at DB level |

### DDL Transactional

```sql
BEGIN;
ALTER TABLE conversations ADD COLUMN category VARCHAR(100);
CREATE INDEX idx_conversations_category ON conversations(category);
COMMIT;
-- ROLLBACK nếu lỗi — MySQL không làm được điều này
```

### Query Highlights

```sql
-- RETURNING — lấy generated values sau INSERT (không cần query riêng)
INSERT INTO conversations (user_id) VALUES ('u1') RETURNING id;

-- Upsert — EXCLUDED = values sẽ được insert nếu không có conflict
INSERT INTO prefs (user_id, key, val) VALUES (%s, %s, %s)
ON CONFLICT (user_id, key) DO UPDATE SET val = EXCLUDED.val;

-- Keyset pagination — không dùng OFFSET
WHERE (created_at, id) < ('2024-06-15 10:30:00', 12345)
ORDER BY created_at DESC, id DESC LIMIT 20;

-- JSONB operators
metadata->>'status'              -- text
metadata->'config'               -- JSON object
metadata @> '{"status": "ok"}'  -- containment (GIN indexable)
metadata ? 'priority'            -- key exists (GIN indexable)
checkpoint_data#>>'{results,0,score}'  -- nested path

-- SQL execution order: FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
-- Alias chỉ dùng được trong ORDER BY trở đi, không dùng trong WHERE
```

### SDK — psycopg v3

```python
# Pool (tạo một lần lúc startup)
pool = ConnectionPool(connection_string, min_size=2, max_size=20)

# Per request
with pool.connection() as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT ... WHERE id = %s", (id,))  # parameterized ALWAYS

# Bulk insert
cur.executemany("INSERT ...", records)              # vài trăm đến vài nghìn rows
with cur.copy("COPY table (...) FROM STDIN") as copy:  # 10K+ rows, 2-10x faster
    for rec in records: copy.write_row(rec)

# Error handling
except UniqueViolation: conn.rollback()
except DeadlockDetected: conn.rollback(); retry()  # No backoff, just retry
except OperationalError: retry_with_exponential_backoff()
```

---

## Module 2: pgvector

### Enable + Schema

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding vector(1536)    -- dimension PHẢI match embedding model
);
```

### Dimensions

| Model | Dimensions |
|---|---|
| Sentence transformers | 384 |
| `text-embedding-ada-002` | **1,536** |
| `text-embedding-3-large` | 3,072 |

### Distance Operators

| Operator | Metric | Recommended |
|---|---|---|
| `<=>` | Cosine distance | ✅ **Text embeddings** (đo góc/direction) |
| `<->` | L2 Euclidean | Khi magnitude quan trọng |
| `<#>` | Negative inner product | Pre-normalized vectors (fastest) |

Cosine: 0 = identical, 2 = opposite. **Smaller = more similar.**

### Vector Types

| Type | Precision | Storage (1536-dim) |
|---|---|---|
| `vector` | 32-bit float | ~6 KB/row |
| `halfvec` | 16-bit float | ~3 KB/row |
| `sparsevec` | Non-zeros only | Varies |

### IVFFlat vs HNSW

| Factor | IVFFlat | HNSW |
|---|---|---|
| Query performance | Good | **Better** |
| Build time | **Faster** | Slower |
| Memory | **Lower** | ~1.5× vector size |
| Empty table | ❌ Cần data trước | ✅ |
| Recall ceiling | Lower | **99%+** |
| Runtime tuning | `ivfflat.probes` | `hnsw.ef_search` |
| Best for | Batch updates, memory-constrained | Production, real-time |

```sql
-- IVFFlat
CREATE INDEX ON docs USING ivfflat (embedding vector_cosine_ops) WITH (lists = 1000);
SET ivfflat.probes = 10;  -- default 1, start sqrt(lists)

-- HNSW
CREATE INDEX ON docs USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);
SET hnsw.ef_search = 100;  -- default 40, 100-200 → 99%+ recall
```

**lists parameter (IVFFlat):**
- ≤100K rows → 100
- 100K–1M → 1,000
- 1M–10M → 4,000–10,000
- >10M → `sqrt(rows)`

### Operator Class — MUST MATCH

| Operator class | Distance operator |
|---|---|
| `vector_cosine_ops` | `<=>` |
| `vector_l2_ops` | `<->` |
| `vector_ip_ops` | `<#>` |

**Mismatch → silent seq scan — no error, just slow.**

### RAG — 2-Table Design

```sql
CREATE TABLE source_documents (id SERIAL PRIMARY KEY, title TEXT, source_url TEXT, ...);
CREATE TABLE document_chunks (
    id SERIAL PRIMARY KEY,
    document_id INTEGER REFERENCES source_documents(id) ON DELETE CASCADE,
    chunk_index INTEGER NOT NULL,
    content TEXT,
    embedding vector(1536),
    token_count INTEGER,
    UNIQUE (document_id, chunk_index)
);

-- Indexes
CREATE INDEX ON document_chunks USING hnsw (embedding vector_cosine_ops);
CREATE INDEX ON document_chunks (document_id);
CREATE INDEX ON document_chunks (document_id, chunk_index);  -- context window queries
```

### Context Window Query Pattern

```sql
WITH matched AS (
    SELECT id, document_id, chunk_index, embedding <=> $1 AS distance
    FROM document_chunks ORDER BY embedding <=> $1 LIMIT 3
),
context AS (
    SELECT DISTINCT dc.*, mc.distance
    FROM matched mc
    JOIN document_chunks dc ON dc.document_id = mc.document_id
        AND dc.chunk_index BETWEEN mc.chunk_index - 1 AND mc.chunk_index + 1
),
budget AS (
    SELECT cw.*, sd.title, sd.source_url,
           SUM(cw.token_count) OVER (ORDER BY cw.distance, cw.chunk_index) AS cumulative_tokens
    FROM context cw JOIN source_documents sd ON cw.document_id = sd.id
)
SELECT * FROM budget WHERE cumulative_tokens <= 3000 ORDER BY distance, chunk_index;
```

### Hybrid Search

```sql
SELECT id, title,
    (1 - (embedding <=> $1)) * 0.7 +
    ts_rank(to_tsvector('english', content), plainto_tsquery('english', $2)) * 0.3
    AS hybrid_score
FROM documents
WHERE to_tsvector('english', content) @@ plainto_tsquery('english', $2)
   OR embedding <=> $1 < 0.5
ORDER BY hybrid_score DESC LIMIT 10;

-- GIN index cho full-text
CREATE INDEX ON documents USING GIN (to_tsvector('english', content));
```

### Storage

- 1536-dim = **~6 KB/row**
- 1M documents = **~6 GB** vector data
- HNSW overhead = **~1.5-2× vector column size**

---

## Module 3: Performance

### Memory Configuration

| Parameter | Purpose | Value |
|---|---|---|
| `shared_buffers` | Shared cache (hot data + indexes) | 25% available memory |
| `work_mem` | Per-operation memory | Set per SESSION/LOCAL, not global |
| `effective_cache_size` | Planner hint (không allocate) | 75% total memory |

**Cache hit ratio < 99% → increase `shared_buffers` first.**

```sql
SELECT sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) AS cache_hit_ratio
FROM pg_statio_user_tables;
```

### Planner Settings (SSD — Azure)

```sql
SET random_page_cost = 1.1;        -- Default 4.0; SSD = random ≈ sequential
SET effective_io_concurrency = 200; -- Default 1; SSD handles parallel I/O
```

### Build Time Estimates

| Index | 1M vectors | 5M vectors | 10M vectors |
|---|---|---|---|
| IVFFlat | 5–15 min | 30–60 min | ~2 hours |
| HNSW (m=16) | 15–45 min | 2–6 hours | 4–12 hours |

### Connection Limits

| Tier | Max connections |
|---|---|
| Burstable 1 vCore | 50 |
| GP 2 vCores | 859 |
| **GP 4 vCores** | **1,718** |
| MO 4 vCores | 3,437 |

New connection overhead: **50-200ms** (TCP + TLS + auth + server process).

### PgBouncer Modes

| Mode | Connection held | Best for |
|---|---|---|
| Session | Entire session | Stateful apps |
| **Transaction** | During transaction only | **Vector search** |
| Statement | Per statement | Max multiplexing |

**Transaction mode: dùng `SET LOCAL`, không phải `SET`** — `SET` không persist sau transaction.

### Scaling Decision Tree

```
Latency cao (single query slow)?
  → Vertical scale: Memory Optimized tier
  → Tăng shared_buffers (cache miss)
  → Check EXPLAIN ANALYZE (seq scan?)

Throughput cao (total queries/sec)?
  → Read replicas (up to 5, async)
  → PgBouncer transaction mode
  → Application ConnectionPool

Connection exhaustion?
  → Enable PgBouncer
  → Reduce pool max_size per instance
  → Pool sizing: server_limit / app_instances - headroom
```

### Monitor Commands

```sql
-- Index usage
SELECT indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes WHERE relname = 'documents';

-- Index build progress
SELECT * FROM pg_stat_progress_create_index;

-- Table + index sizes
SELECT pg_size_pretty(pg_total_relation_size('documents'));

-- Replica lag (run on replica)
SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS lag_seconds;

-- Verify index in use
EXPLAIN ANALYZE SELECT ... ORDER BY embedding <=> $1 LIMIT 10;
-- Good: "Index Scan using documents_embedding_idx"
-- Bad: "Seq Scan" → check operator class, LIMIT present?, index exists?
```

---

## CLI Quick Reference

### az — Azure CLI

```bash
# Tạo server
az postgres flexible-server create \
    --resource-group myRG --name myserver \
    --location eastus --tier GeneralPurpose --sku-name Standard_D4s_v3 \
    --storage-size 128 --backup-retention 14

# Enable PgBouncer
az postgres flexible-server parameter set \
    --resource-group myRG --server-name myserver \
    --name pgbouncer.enabled --value true

# Set parameter
az postgres flexible-server parameter set \
    --resource-group myRG --server-name myserver \
    --name random_page_cost --value 1.1

# Create read replica
az postgres flexible-server replica create \
    --resource-group myRG --replica-name myserver-replica \
    --source-server myserver

# Scale compute
az postgres flexible-server update \
    --resource-group myRG --name myserver \
    --sku-name Standard_E4s_v3 --tier MemoryOptimized

# Show connection info
az postgres flexible-server show-connection-string --server-name myserver
```

### psql — Connect & Operate

```bash
# Connect
psql "host=myserver.postgres.database.azure.com dbname=mydb user=myuser sslmode=require"

# Via PgBouncer
psql "host=myserver.postgres.database.azure.com port=6432 dbname=mydb user=myuser sslmode=require"

# Enable extension
psql -c "CREATE EXTENSION IF NOT EXISTS vector;"

# Check extensions
psql -c "SELECT * FROM pg_extension;"

# Run file
psql -f schema.sql

# REINDEX
psql -c "REINDEX INDEX CONCURRENTLY documents_embedding_idx;"
```

---

## Exam Traps — Tối thiểu 10 điểm

1. **PgBouncer tier restriction:** PgBouncer chỉ có General Purpose và Memory Optimized. Burstable → không có PgBouncer → phải dùng application-level pool.

2. **IVFFlat cần data trước khi tạo index:** Empty table → k-means không train được → index tạo thành công nhưng unusable. HNSW OK với empty table.

3. **Operator class mismatch = silent seq scan:** Không có error, query trả kết quả đúng nhưng qua sequential scan. `EXPLAIN ANALYZE` là cách duy nhất phát hiện.

4. **RETURNING vs OUTPUT vs LAST_INSERT_ID():** PostgreSQL = `RETURNING`. SQL Server = `OUTPUT`. MySQL = `LAST_INSERT_ID()`. Cross-DB syntax error nhưng logic đúng.

5. **Point-in-time restore tạo server MỚI:** Không restore in-place. App phải update connection string đến server mới. Khác với SQL Server restore database in-place.

6. **Latency vs Throughput scaling:** Read replicas giải quyết throughput (total queries/sec), không giải quyết latency (single query slow). Vertical scale (Memory Optimized) giải quyết latency.

7. **work_mem là per-operation per-connection:** `SET work_mem = '256MB'` globally với 100 connections × 5 sort ops = 128 GB potential. Luôn dùng `SET LOCAL` trong transaction hoặc `SET` per session.

8. **`ef_construction` vs `ef_search` — build time vs query time:** `ef_construction` ảnh hưởng index quality khi build, không ảnh hưởng query speed. Tăng recall mà không rebuild → tăng `ef_search`.

9. **PgBouncer transaction mode và `SET`:** `SET hnsw.ef_search = 200` không persist sau transaction trong transaction mode. Phải dùng `SET LOCAL hnsw.ef_search = 200` trong mỗi transaction.

10. **DDL transactional chỉ trong PostgreSQL:** `BEGIN; ALTER TABLE; CREATE INDEX; COMMIT;` có thể rollback nếu lỗi. MySQL và SQL Server auto-commit DDL — không thể rollback ALTER TABLE.

11. **Cosine distance threshold direction:** `WHERE embedding <=> $1 < 0.4` = more similar (distance NHỎ = similar hơn). Không phải `> 0.4`. Confusion với similarity (1 - distance) score.

12. **IVFFlat `lists` formula:** ≤1M rows = `rows/1000`, >1M rows = `sqrt(rows)`. Exam thường cho 5M rows → `sqrt(5000000) ≈ 2236`, không phải `5000000/1000 = 5000`.

13. **VARCHAR(MAX) không tồn tại trong PostgreSQL:** SQL Server syntax. PostgreSQL dùng `TEXT` (unlimited) hoặc `VARCHAR(n)`.

14. **ON DUPLICATE KEY UPDATE là MySQL:** PostgreSQL = `ON CONFLICT (col) DO UPDATE SET col = EXCLUDED.col`. MySQL = `ON DUPLICATE KEY UPDATE col = VALUES(col)`.

15. **`(xmax = 0)` detect insert vs update:** `RETURNING id, (xmax = 0) AS is_new` — true khi row vừa được INSERT, false khi UPDATE. Không có equivalent ở MySQL hay SQL Server.

---

## Assessment Summaries

### Module 1 Assessment — 5/5

| Câu | Câu hỏi | Đáp án |
|---|---|---|
| 1 | Varying structure metadata | **JSONB** (không phải TEXT, không phải VARCHAR(MAX)) |
| 2 | Get generated ID sau INSERT | **RETURNING** (không phải OUTPUT/LAST_INSERT_ID) |
| 3 | Insert or update nếu exists | **ON CONFLICT DO UPDATE** (không phải ON DUPLICATE KEY) |
| 4 | Many short-lived connections | **ConnectionPool** (không phải new conn/global conn) |
| 5 | Enforce allowed status values | **CHECK (status IN (...))** (không phải UNIQUE/NOT NULL) |

### Module 2 Assessment — 5/5

| Câu | Câu hỏi | Đáp án |
|---|---|---|
| 1 | Normalized embeddings, semantic similarity | **`<=>` cosine** (không phải `<->` L2) |
| 2 | 5M embeddings, occasional batch updates | **IVFFlat** `lists=sqrt(n)` (không phải HNSW — build time) |
| 3 | HNSW `m` parameter controls | **Max connections per node** (không phải ef_construction/lists) |
| 4 | Update 50K embeddings, minimize impact | **Batch 1,000–5,000 rows** (không phải single txn/drop index) |
| 5 | Balance vector + full-text scores | **RRF (Reciprocal Rank Fusion)** (không phải multiply/always vector first) |

### Module 3 Assessment — 5/5

| Câu | Câu hỏi | Đáp án |
|---|---|---|
| 1 | Cache hit ratio 85%, slow queries | **Tăng `shared_buffers`** (không phải random_page_cost/probes) |
| 2 | 5M vectors, daily refresh, build <30 min | **IVFFlat** `lists=sqrt(rows)` (HNSW = 2-6 hours) |
| 3 | Seq scan với category filter | **Verify B-tree index trên `category_id`** (không phải vector operator class) |
| 4 | 500 queries/sec, 1,719 max connections | **PgBouncer transaction mode** (không phải new conn/session mode) |
| 5 | CPU 75%, P95 150ms, cần <50ms | **Upgrade Memory Optimized** (không phải replicas/Redis) |

---

## So sánh PostgreSQL vs Cosmos DB for NoSQL vs Redis

| Dimension | Azure DB for PostgreSQL | Cosmos DB for NoSQL | Azure Cache for Redis |
|---|---|---|---|
| Data model | Relational + JSONB + vector | Document (JSON) | Key-value, sorted sets, hashes |
| Query language | SQL (full) | SQL API (subset) | Redis commands |
| ACID transactions | ✅ Full | ✅ (single partition) | ✅ (single shard, MULTI/EXEC) |
| Vector search | ✅ pgvector (IVFFlat/HNSW) | ❌ | ✅ RediSearch (HNSW) |
| Schema enforcement | ✅ Strict types + constraints | ❌ Schemaless | ❌ Schemaless |
| Relational JOIN | ✅ Native | ❌ | ❌ |
| Global distribution | ❌ Single region + replicas | ✅ Multi-region active-active | ✅ (Azure Cache geo-replication) |
| Horizontal sharding | ❌ (Citus = separate product) | ✅ Automatic | ✅ Cluster mode |
| Max query latency | Milliseconds (indexed) | <10ms (key) | <1ms |
| Connection protocol | PostgreSQL wire | HTTPS/TCP | Redis protocol |
| Python driver | psycopg v3 | azure-cosmos | redis-py |
| Full-text search | ✅ `tsvector/tsquery` | ❌ | ✅ RediSearch |
| Backup/PITR | ✅ 7–35 days | ✅ Continuous | ✅ RDB/AOF |
| Pricing model | vCore + storage | RU/s (provisioned/serverless) | Cache size |

### Khi nào dùng cái nào?

**Dùng PostgreSQL khi:**
- Cần relational integrity (FK, JOIN, transactions)
- AI app có conversation history + vector search trong cùng schema
- RAG với complex metadata filtering
- Audit trail, compliance, ACID bắt buộc
- Team biết SQL

**Dùng Cosmos DB for NoSQL khi:**
- Global distribution bắt buộc (multi-region active-active)
- Schemaless documents với variable structure
- Key-based access dominant (<10ms latency target)
- Massive scale với automatic sharding
- Không cần vector search (Cosmos không có pgvector)

**Dùng Redis khi:**
- Caching layer cho repeated queries (session, embedding cache)
- Real-time leaderboards, counters, pub/sub
- Sub-millisecond latency cho simple key lookups
- Complement PostgreSQL (không thay thế)
- RediSearch khi cần vector search + extremely low latency (<1ms)

---

*Tổng hợp AI-200 · Azure Database for PostgreSQL · 3 Modules · 12 Lessons*
