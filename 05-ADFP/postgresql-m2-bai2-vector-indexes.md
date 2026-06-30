# Bài 2 — Perform Fast Vector Similarity Search (Indexes)

> Khoá: AI-200 · PostgreSQL — Implement vector search with Azure Database for PostgreSQL

---

## Exact vs Approximate Search

**Exact search (no index):** So sánh với mọi row → kết quả 100% chính xác nhưng O(n) → chậm với large datasets.

**Approximate Nearest Neighbor (ANN):** Trade accuracy for speed. **Recall** = % of true nearest neighbors returned.

95-99% recall thường đủ cho AI application — speed trade-off worthwhile.

---

## IVFFlat Index

Partitions vectors vào **clusters** lúc build. Query chỉ tìm trong N clusters gần nhất.

```sql
CREATE INDEX documents_embedding_idx ON documents
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);    -- Số clusters
```

**lists parameter:**
- ≤1M rows: `rows / 1000`
- >1M rows: `sqrt(rows)`

**probes** (query time): số clusters search
```sql
SET ivfflat.probes = 10;    -- Mặc định 1, higher = better recall + slower
-- Start: probes = sqrt(lists)
```

> **IVFFlat cần data trước khi tạo index.** Empty table → index unusable. Load data first, then create index.

---

## HNSW Index

Multi-layer graph structure. Từng layer có connections đến nearest neighbors.

```sql
CREATE INDEX documents_embedding_idx ON documents
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

- `m` = max connections per node (default 16, 32-64 cho high recall)
- `ef_construction` = search width lúc build (default 64, 100-200 cho production quality)

```sql
SET hnsw.ef_search = 100;    -- Search width lúc query, default 40
-- 100-200 đạt 99%+ recall
```

**HNSW có thể tạo trên empty table** — graph updates incrementally khi insert.

---

## IVFFlat vs HNSW

| Factor | IVFFlat | HNSW |
|---|---|---|
| Query performance | Good | **Better** |
| Build time | Faster | Slower |
| Memory usage | Lower | Higher |
| Empty table support | ❌ | ✅ |
| Insert performance | Fast | Moderate |
| Recall at same latency | Lower | Higher |

**Chọn HNSW** cho hầu hết production AI app — query performance và recall tốt hơn.

**Chọn IVFFlat** khi: memory constraint, faster builds, data thay đổi thường xuyên.

---

## Operator Class phải match Distance Operator

| Operator class | Distance operator | Distance metric |
|---|---|---|
| `vector_l2_ops` | `<->` | L2 (Euclidean) |
| `vector_cosine_ops` | `<=>` | Cosine |
| `vector_ip_ops` | `<#>` | Inner product |

**Không match → PostgreSQL không dùng index → fall back to sequential scan.**

---

## Verify Index Usage

```sql
EXPLAIN ANALYZE
SELECT id, title
FROM documents
ORDER BY embedding <=> '[0.0123, -0.0456, ...]'::vector
LIMIT 10;
```

Output tốt: `Index Scan using documents_embedding_idx`
Output xấu: `Seq Scan` → kiểm tra operator class match, index tồn tại, table đủ rows, có LIMIT

---

## Bản chất bài này là gì?

**Một câu:** IVFFlat và HNSW là hai ANN algorithms với trade-off rõ ràng — IVFFlat cho batch-update workloads với memory constraint, HNSW cho production real-time với recall cao — và operator class phải khớp distance operator nếu không PostgreSQL silently fall back to seq scan.

### So sánh với Faiss (Meta) vs ScaNN (Google) vs Annoy (Spotify)

| | pgvector HNSW | pgvector IVFFlat | Faiss HNSW | Faiss IVF | Annoy |
|---|---|---|---|---|---|
| Build on empty table | ✅ | ❌ (cần data) | ✅ | ❌ | ✅ |
| Incremental inserts | ✅ | Degraded recall | ✅ | Degraded | ❌ (read-only after build) |
| Query tunability | `ef_search` (no rebuild) | `probes` (no rebuild) | `ef` (no rebuild) | `nprobe` | `search_k` |
| Memory per vector (1536-dim) | ~1.5x data | ~1x data | ~1.5x data | ~1x data | ~1x data |
| SQL integration | ✅ Native | ✅ Native | ❌ Standalone | ❌ Standalone | ❌ Standalone |
| Concurrent build | `CREATE INDEX CONCURRENTLY` | `CREATE INDEX CONCURRENTLY` | ❌ | ❌ | ❌ |

**Operator class mismatch là silent failure — đây là exam trap số 1:** PostgreSQL không throw error khi operator class không khớp với distance operator. Query vẫn chạy, vẫn trả kết quả đúng — nhưng dùng sequential scan thay vì index. Với 1M vectors, sequential scan mất seconds thay vì milliseconds. `EXPLAIN ANALYZE` là cách duy nhất phát hiện.

**IVFFlat "cần data trước khi tạo index" khác với Faiss IVF cơ bản:** IVFFlat dùng k-means clustering trên data hiện có để tạo centroids. Empty table = không có data để cluster = index tạo thành công nhưng không có centroids = không có gì để search. HNSW graph construction thêm từng vector vào graph incrementally nên empty table hoàn toàn OK.

---

## Checklist ghi nhớ cho AI-200

- [ ] **IVFFlat:** `lists` = rows/1000 (≤1M) · `probes` = query-time search width
- [ ] IVFFlat cần **data trước khi tạo index** — empty table → unusable
- [ ] **HNSW:** `m` = connections per node · `ef_construction` = build quality
- [ ] HNSW có thể tạo trên **empty table**
- [ ] HNSW `ef_search` (query time, default 40) — tăng để tăng recall
- [ ] **Operator class phải match distance operator** — sai = seq scan
- [ ] HNSW recommended cho production (query performance + recall tốt hơn)
- [ ] `EXPLAIN ANALYZE` để verify index đang được dùng
- [ ] LIMIT clause là cần thiết để index được dùng

---

[← Bài 1](./postgresql-m2-bai1-pgvector-basics.md) · [🏠 Mục lục](../README.md) · [Bài 3 →](./postgresql-m2-bai3-lifecycle-retrieval.md)
