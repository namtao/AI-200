# Bài 1 — Tune PostgreSQL + Choose Vector Indexes

> Khoá: AI-200 · PostgreSQL — Optimize vector search in Azure Database for PostgreSQL

---

## Compute & Memory Requirements

Vector search khác với traditional SQL: mỗi query có thể cần tính distance với millions of vectors.

**1536-dim vector = 1,536 floating-point ops per distance calculation**
→ 1M vectors without index = 1.5 billion ops per query = seconds.

**Distance operator cost (ascending):**
- `<->` L2 (Euclidean) — most expensive (sqrt of sum of squares)
- `<=>` Cosine — middle (normalizes internally)
- `<#>` Inner product — fastest (dot product, cần pre-normalized vectors)

**Storage:**

| Dimensions | Bytes/vector | 1M vectors |
|---|---|---|
| 384 | ~1.5 KB | ~1.5 GB |
| 768 | ~3 KB | ~3 GB |
| **1536** | **~6 KB** | **~6 GB** |
| 3072 | ~12 KB | ~12 GB |

---

## Memory Configuration

### shared_buffers — Shared cache

Giữ vector indexes và hot data trong memory. Cache hit ratio < 99% = `shared_buffers` quá nhỏ.

```sql
SHOW shared_buffers;

-- Check cache hit ratio
SELECT
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) AS cache_hit_ratio
FROM pg_statio_user_tables;
```

Starting point: 25% available memory. Azure auto-tunes theo compute tier.

### work_mem — Per-operation memory

Vector queries sorting large results cần `work_mem` lớn hơn default (4 MB).

```sql
-- Set per session/transaction (không phải globally)
SET work_mem = '256MB';
-- Hoặc trong transaction
SET LOCAL work_mem = '256MB';
```

**Cảnh báo:** `work_mem` áp dụng per-operation per-connection. 100 connections × complex queries = nhiều memory.

### effective_cache_size — Planner hint

Cho planner biết tổng available cache (shared_buffers + OS file cache). Không allocate memory — chỉ là hint.

```
effective_cache_size ≈ 75% total memory (dedicated server)
```

Higher → planner prefer index scans → tốt cho vector search.

---

## Query Planner Settings

| Parameter | Default | Recommended (SSD) | Effect |
|---|---|---|---|
| `random_page_cost` | 4.0 | **1.1-1.5** | Encourage index scans (SSD = random access ≈ sequential) |
| `effective_io_concurrency` | 1 | **200** | Prefetch pages in parallel (SSD handles concurrent I/O) |

```sql
SET random_page_cost = 1.1;
```

---

## pgvector Parameters

### IVFFlat — `ivfflat.probes`

```sql
SET ivfflat.probes = 10;    -- Default 1, higher = better recall + slower
-- Start: sqrt(lists), adjust based on measured recall
```

### HNSW — `hnsw.ef_search`

```sql
SET hnsw.ef_search = 100;    -- Default 40
-- 100-200 → 99%+ recall
-- 20 → faster, approximate OK
```

---

## Index Selection

### IVFFlat Parameters

```sql
CREATE INDEX ON products
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 1000);
```

**lists parameter:**
- ≤100K rows → 100 lists
- 100K–1M rows → 1,000 lists
- 1M–10M rows → 4,000-10,000 lists
- >10M rows → `sqrt(rows)`

> IVFFlat cần data trước khi tạo index (training on empty table = unusable).

### HNSW Parameters

```sql
CREATE INDEX ON products
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

- `m` = connections per node (default 16, 32-64 cho high recall)
- `ef_construction` = build quality (min 2×m, production 4×m+)
- Có thể tạo trên empty table, updates incrementally

### Comparison

| | IVFFlat | HNSW |
|---|---|---|
| Recall | Lower | **Higher (99%+)** |
| Build time | Faster | Slower |
| Memory | Lower | Higher (~1.5x vector size) |
| Empty table | ❌ | ✅ |
| Insert performance | Fast | Moderate |

**Choose HNSW:** Query performance priority, high recall, mostly read.
**Choose IVFFlat:** Memory constrained, frequent bulk updates, faster builds.
**Azure DiskANN:** Enterprise-scale (100M+ vectors) — Azure-specific feature.

### Table size vs Index benefit

| Size | Benefit |
|---|---|
| <10K | Limited (sequential scan OK) |
| 10K-100K | Moderate |
| 100K-1M | Significant — index required |
| **>1M** | **Essential** |

---

## Monitor với EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE
SELECT id, title
FROM products
ORDER BY embedding <=> $1
LIMIT 10;
```

**Good:** `Index Scan using documents_embedding_idx`
**Bad:** `Seq Scan` — kiểm tra: operator class match? index tồn tại? LIMIT có không?

---

## Checklist ghi nhớ cho AI-200

- [ ] Cache hit ratio < 99% → tăng `shared_buffers`
- [ ] `work_mem` = per-operation per-connection → set SESSION, không global
- [ ] `random_page_cost = 1.1` cho Azure SSD storage
- [ ] `effective_io_concurrency = 200` cho SSD
- [ ] IVFFlat `lists`: rows/1000 (≤1M) · probes: start `sqrt(lists)`
- [ ] HNSW `m` = connections per node · `ef_construction` = build quality (min 2×m)
- [ ] IVFFlat cần data trước, HNSW OK với empty table
- [ ] HNSW recommended cho production (recall + query performance)
- [ ] Operator class phải match distance operator
- [ ] `EXPLAIN ANALYZE` để verify index usage

---

*Bài tiếp theo: Bài 2 — Choose and configure vector indexes**
