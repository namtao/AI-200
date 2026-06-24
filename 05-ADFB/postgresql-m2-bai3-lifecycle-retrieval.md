# Bài 3 — Index Lifecycle + Semantic Retrieval Patterns

> Khoá: AI-200 · PostgreSQL — Implement vector search with Azure Database for PostgreSQL

---

## Monitor Index Health

```sql
-- Xem index usage và size
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan AS times_used,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE indexrelname LIKE '%embedding%'
ORDER BY idx_scan DESC;
```

`idx_scan = 0` → queries không dùng index → kiểm tra operator class match.

Monitor build progress:
```sql
SELECT * FROM pg_stat_progress_create_index;
```

---

## Rebuild Index khi Data Distribution Thay Đổi

ANN indexes optimize based on data distribution at creation time. Signals cần rebuild:
- Query latency tăng không tương ứng với data growth
- User report kém relevant results
- Thêm >20-30% data mới từ different domain

**Production — không downtime:**
```sql
-- Tạo index mới concurrent (không block queries)
CREATE INDEX CONCURRENTLY documents_embedding_new_idx
ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Verify rồi swap
DROP INDEX documents_embedding_idx;
ALTER INDEX documents_embedding_new_idx RENAME TO documents_embedding_idx;
```

**Có thể tolerate downtime:**
```sql
REINDEX INDEX documents_embedding_idx;    -- Blocks writes
```

---

## Embedding Updates — Stale Flag Pattern

```sql
-- Thêm column tracking
ALTER TABLE documents ADD COLUMN embedding_stale BOOLEAN DEFAULT FALSE;

-- Content update (fast, không gọi embedding API)
UPDATE documents
SET content = 'New content...', embedding_stale = TRUE, updated_at = NOW()
WHERE id = 42;

-- Background job: fetch stale, generate embeddings, update
SELECT id, content FROM documents WHERE embedding_stale = TRUE LIMIT 100;

-- Bulk update sau khi generate
UPDATE documents
SET embedding = data.embedding, embedding_stale = FALSE
FROM (VALUES (1, '[...]'::vector), (2, '[...]'::vector)) AS data(id, embedding)
WHERE documents.id = data.id;
```

Batch approach: decouples content update from embedding API call → lower latency, more resilient.

---

## Embedding Model Migration — Parallel Column Strategy

Không thể mix embeddings từ different models trong cùng column.

```sql
-- Bước 1: Add column mới
ALTER TABLE documents ADD COLUMN embedding_v2 vector(3072);

-- Bước 2: Create index trên column mới
CREATE INDEX documents_embedding_v2_idx
ON documents USING hnsw (embedding_v2 vector_cosine_ops);

-- Bước 3: Backfill theo batches
UPDATE documents SET embedding_v2 = '[...]'::vector
WHERE id BETWEEN 1 AND 10000 AND embedding_v2 IS NULL;

-- Bước 4: Test và validate
SELECT id, title,
       embedding <=> $1 AS v1_distance,
       embedding_v2 <=> $2 AS v2_distance
FROM documents ORDER BY embedding_v2 <=> $2 LIMIT 10;

-- Bước 5: Cleanup
DROP INDEX documents_embedding_idx;
ALTER TABLE documents DROP COLUMN embedding;
ALTER TABLE documents RENAME COLUMN embedding_v2 TO embedding;
ALTER INDEX documents_embedding_v2_idx RENAME TO documents_embedding_idx;
```

---

## Vector Similarity Search với Metadata Filters

```sql
-- Filter trước vector math
SELECT id, title, embedding <=> $1 AS distance
FROM documents
WHERE category = 'contracts'
  AND practice_area = 'corporate'
  AND created_at > '2024-01-01'
ORDER BY embedding <=> $1
LIMIT 10;
```

Index B-tree trên filter columns:
```sql
CREATE INDEX idx_documents_category ON documents (category);
CREATE INDEX idx_documents_category_practice ON documents (category, practice_area);
```

`ANALYZE` sau khi load data để planner có statistics chính xác.

---

## Distance Threshold

```sql
-- Chỉ return nếu đủ similar (cosine distance < 0.4)
SELECT id, title, embedding <=> $1 AS distance
FROM documents
WHERE embedding <=> $1 < 0.4    -- Quality floor
ORDER BY embedding <=> $1
LIMIT 10;
```

Cosine thresholds thường: 0.3 (strict, high precision) đến 0.6 (lenient, high recall).

---

## Hybrid Search (Vector + Full-text)

```sql
SELECT
    id,
    title,
    (1 - (embedding <=> $1)) * 0.7 +
        ts_rank(to_tsvector('english', content), plainto_tsquery('english', $2)) * 0.3
        AS hybrid_score
FROM documents
WHERE to_tsvector('english', content) @@ plainto_tsquery('english', $2)
   OR embedding <=> $1 < 0.5
ORDER BY hybrid_score DESC
LIMIT 10;
```

Weight 0.7/0.3: semantic dominates. Điều chỉnh based on use case.

GIN index cho full-text:
```sql
CREATE INDEX idx_documents_content_fts ON documents USING GIN (to_tsvector('english', content));
```

---

## Storage và VACUUM

1536-dim vector = **~6 KB/row**. 1M documents = ~6 GB vector data.

HNSW overhead: ~1.5-2x vector column size.

```sql
-- Check sizes
SELECT
    pg_size_pretty(pg_relation_size('documents')) AS table_size,
    pg_size_pretty(pg_indexes_size('documents')) AS index_size,
    pg_size_pretty(pg_total_relation_size('documents')) AS total_size;

-- Reclaim space (MVCC dead tuples)
VACUUM documents;

-- Autovacuum more aggressive cho heavy update tables
ALTER TABLE documents SET (
    autovacuum_vacuum_scale_factor = 0.05,   -- Default 0.20
    autovacuum_analyze_scale_factor = 0.02   -- Default 0.10
);
```

---

## Checklist ghi nhớ cho AI-200

- [ ] `CREATE INDEX CONCURRENTLY` → rebuild không block queries
- [ ] Stale flag pattern: mark content update → background job generate + update embedding
- [ ] Model migration: parallel column → backfill → validate → swap → cleanup
- [ ] Filter columns cần B-tree index để planner estimate selectivity
- [ ] `ANALYZE` sau khi load data → planner statistics
- [ ] Distance threshold: `WHERE embedding <=> $1 < 0.4` → quality floor
- [ ] Hybrid search: `(1 - cosine_distance) * weight + ts_rank * weight`
- [ ] GIN index cho full-text search
- [ ] 1536-dim = ~6 KB/row · HNSW overhead ~1.5-2x vector size
- [ ] VACUUM để reclaim space từ MVCC dead tuples
- [ ] `autovacuum_vacuum_scale_factor = 0.05` cho heavy update tables

---

*Bài tiếp theo: Bài 4 — RAG Retrieval Patterns*
