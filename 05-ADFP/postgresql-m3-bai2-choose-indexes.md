# Bài 2 — Choose and Configure Vector Indexes

> Khoá: AI-200 · PostgreSQL — Optimize vector search in Azure Database for PostgreSQL

---

## Exact vs Approximate Search

| | Exact (no index) | ANN (indexed) |
|---|---|---|
| Accuracy | 100% (true nearest neighbors) | ~95-99% recall |
| Speed | O(n) — grows with table size | O(log n) or sublinear |
| Dùng khi | <10K rows, need guaranteed exact | Production scale |

**Recall** = fraction of true nearest neighbors returned. 95-99% thường đủ cho AI app — speed improvement 100-1000× worthwhile.

### Table size vs Index benefit

| Size | Benefit |
|---|---|
| <10K rows | Limited |
| 10K–100K | Moderate |
| 100K–1M | Significant — required for interactive use |
| **>1M rows** | **Essential** |

---

## IVFFlat Index

K-means clustering → partitions vectors vào `lists` clusters.
Query → find nearest cluster centroids → search chỉ trong `probes` clusters.

```sql
CREATE INDEX ON products
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 1000);
```

**lists parameter:**

| Rows | Lists |
|---|---|
| ≤100K | 100 |
| 100K–1M | 1,000 |
| 1M–10M | 4,000–10,000 |
| >10M | `sqrt(rows)` |

```sql
SET ivfflat.probes = 20;    -- How many clusters to search
-- Start: sqrt(lists), adjust based on recall
```

**Đặc điểm:**
- Fast build time
- Lower memory
- Cần data trước khi tạo (empty table → unusable)
- Recall ceiling thấp hơn HNSW
- Cần rebuild khi data distribution thay đổi nhiều

---

## HNSW Index

Multi-layer graph. Coarse-to-fine navigation: top layers ít nodes → bottom layer tất cả nodes.

```sql
CREATE INDEX ON products
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

- `m` = max connections per node (default 16, tăng 32-64 cho high recall)
- `ef_construction` = build quality (tối thiểu 2×m, production 4×m+)
- Không ảnh hưởng query time — chỉ build time và graph quality

```sql
SET hnsw.ef_search = 100;    -- Default 40
-- 100-200 → 99%+ recall
```

**Đặc điểm:**
- Better query performance và recall (99%+)
- Higher memory (~1.5× vector data for m=16)
- Longer build time
- **Có thể tạo trên empty table** — incremental updates
- Consistent performance regardless of data distribution

---

## IVFFlat vs HNSW Comparison

| Factor | IVFFlat | HNSW |
|---|---|---|
| Query performance | Good | **Better** |
| Build time | **Faster** | Slower |
| Memory | **Lower** | Higher |
| Empty table | ❌ | ✅ |
| Recall | Lower | **99%+** |
| Runtime tuning | `probes` without rebuild | `ef_search` without rebuild |

**Chọn HNSW:** Query performance priority, high recall, mostly reads.
**Chọn IVFFlat:** Memory limited, frequent bulk rebuilds, faster build needed.
**Azure DiskANN:** Enterprise scale (100M+ vectors) — Azure-specific.

---

## Operator Class phải match Distance Operator

| Operator class | Distance operator | Metric |
|---|---|---|
| `vector_cosine_ops` | `<=>` | Cosine |
| `vector_l2_ops` | `<->` | L2 (Euclidean) |
| `vector_ip_ops` | `<#>` | Inner product |

**Mismatch → PostgreSQL không dùng index → sequential scan.**

---

## Create, Monitor, Maintain

```sql
-- Concurrent build (không block writes)
CREATE INDEX CONCURRENTLY idx_products_embedding
ON products USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Monitor build progress
SELECT phase, blocks_total, blocks_done, tuples_total, tuples_done
FROM pg_stat_progress_create_index;

-- Check index size và usage
SELECT indexrelname, pg_size_pretty(pg_relation_size(indexrelid)) AS size, idx_scan
FROM pg_stat_user_indexes WHERE relname = 'products';

-- Reindex khi cần
REINDEX INDEX CONCURRENTLY idx_products_embedding;
```

**Build time estimates:**
- IVFFlat 1M vectors: 5–15 phút
- HNSW 1M vectors: 15–45 phút
- HNSW 10M vectors: 2–6 giờ

---

## Bản chất bài này là gì?

**Một câu:** Chọn IVFFlat hay HNSW là quyết định dựa trên 3 yếu tố — build time constraint, memory budget, và insert pattern (batch vs real-time) — không phải chỉ "HNSW tốt hơn" mà là tùy workload.

### So sánh với Weaviate HNSW vs Qdrant HNSW vs Faiss IVF (decision framework)

| Scenario | pgvector recommendation | Reasoning |
|---|---|---|
| 5M vectors, daily batch rebuild, <30 min build | **IVFFlat** `lists=sqrt(n)` | Build time constraint; batch = OK to rebuild |
| 1M vectors, real-time inserts, 99% recall needed | **HNSW** `m=16, ef_c=64` | Incremental insert, high recall |
| 100M+ vectors, Azure production | **Azure DiskANN** (Azure-specific) | Graph-based, disk-optimized, enterprise scale |
| <10K vectors, dev/test | **No index** (exact scan) | Index overhead not worth it |
| Memory-constrained server | **IVFFlat** | Lower memory footprint |
| Unknown pattern, start conservative | **HNSW** `m=16, ef_c=64, ef_s=40` | Default production-safe |

| Build time estimate | IVFFlat | HNSW |
|---|---|---|
| 1M vectors | 5-15 min | 15-45 min |
| 5M vectors | 30-60 min | **2-6 hours** |
| 10M vectors | ~2 hours | **4-12 hours** |

**Azure DiskANN không có trong pgvector open-source:** DiskANN là Microsoft Research algorithm được tích hợp vào Azure Database for PostgreSQL Flexible Server nhưng không available trong vanilla pgvector. Weaviate và Qdrant standalone cũng không có DiskANN. Đây là Azure-specific competitive advantage tại 100M+ scale.

**HNSW `ef_construction` ảnh hưởng index quality (build-time only), không ảnh hưởng query speed:** Tăng `ef_construction` từ 64 → 200 → build lâu hơn, index quality cao hơn → nhưng query speed không đổi. Query speed chỉ phụ thuộc `ef_search`. Exam thường hỏi: "tăng tham số nào để cải thiện recall mà không rebuild?" → `ef_search` (runtime), không phải `ef_construction` (build-time).

---

## Checklist ghi nhớ cho AI-200

- [ ] ANN index: 95-99% recall, 100–1000× faster than exact search
- [ ] IVFFlat: k-means clustering, `lists`, `probes` at query time
- [ ] IVFFlat cần **data trước khi tạo index**
- [ ] HNSW: graph structure, `m` + `ef_construction` (build), `ef_search` (query)
- [ ] HNSW tạo được trên **empty table**, incremental updates
- [ ] HNSW `m=16, ef_construction=64` default → `ef_search=40` default
- [ ] Operator class phải khớp với distance operator — sai → seq scan
- [ ] `CREATE INDEX CONCURRENTLY` để không block writes
- [ ] HNSW 5M vectors build ≈ 2-6 giờ

---

[← Bài 1](./postgresql-m3-bai1-tune-indexes.md) · [🏠 Mục lục](../README.md) · [Bài 3 →](./postgresql-m3-bai3-data-layout.md)
