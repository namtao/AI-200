# Module Assessment — Implement Vector Search with Azure Database for PostgreSQL

> Khoá: AI-200 · PostgreSQL — Implement vector search with Azure Database for PostgreSQL

---

## Câu 1

**Which pgvector distance operator should you use when your embeddings are normalized to unit length and you want to measure semantic similarity?**

- `<=>` (cosine distance) ✅
- `<->` (L2 distance)
- `<#>` (negative inner product)

**Đáp án: `<=>` cosine distance**

Cosine distance đo **góc** giữa hai vectors, bỏ qua magnitude. Với normalized embeddings (unit length), cosine distance phù hợp nhất vì:
- Text embeddings capture **direction of meaning**, không phải magnitude
- Azure OpenAI và hầu hết text embedding models được optimize cho cosine similarity
- Score 0 = identical direction, 2 = opposite directions

```sql
SELECT id, title, embedding <=> $1 AS distance
FROM documents
ORDER BY embedding <=> $1    -- Ascending: most similar first
LIMIT 10;
```

Index phải dùng `vector_cosine_ops`:
```sql
CREATE INDEX ... USING hnsw (embedding vector_cosine_ops)
```

Tại sao các đáp án còn lại sai:
- **`<->` L2 distance:** Đo khoảng cách Euclidean (straight-line distance) — phụ thuộc vào magnitude. Với normalized embeddings, L2 và cosine tương quan nhau, nhưng cosine là conventional choice và được document rõ hơn cho text similarity.
- **`<#>` negative inner product:** Dot product với magnitude ảnh hưởng. Cho normalized vectors, mathematically equivalent với cosine nhưng ít conventional hơn. Câu hỏi hỏi về "semantic similarity" — cosine là answer mong đợi.

---

## Câu 2

**You're building a RAG pipeline that needs to retrieve relevant document chunks quickly from a collection of 5 million embeddings. The collection receives occasional batch updates but no real-time inserts. Which index type should you choose?**

- IVFFlat with an appropriate number of lists ✅
- HNSW with high ef_construction value
- No index, relying on exact sequential scan

**Đáp án: IVFFlat với appropriate number of lists**

Dataset có "occasional batch updates, no real-time inserts" — đây là điều kiện lý tưởng cho IVFFlat:
- **No real-time inserts** = không cần HNSW's strength (incremental insert performance). Với IVFFlat, có thể rebuild index sau mỗi batch update.
- **Memory efficiency**: IVFFlat tốn ít RAM hơn HNSW đáng kể ở 5M vectors — HNSW cần lưu toàn bộ graph in memory.
- **Appropriate lists**: `sqrt(5000000) ≈ 2236` lists — trade-off giữa recall và speed có thể tune được qua `ivfflat.probes`.

```sql
CREATE INDEX doc_chunks_embedding_idx ON document_chunks
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 2236);    -- sqrt(n) là starting point

SET ivfflat.probes = 10;    -- Tăng số lists được scan để tăng recall
```

Tại sao các đáp án còn lại sai:
- **HNSW:** Tốt hơn khi cần real-time inserts và có đủ RAM. Với 5M vectors và no real-time requirement, HNSW's overhead về memory không justified. IVFFlat phù hợp hơn khi dataset có thể batch-rebuild index.
- **No index (sequential scan):** 5 million distance calculations per query = seconds per query. Completely unsuitable cho RAG pipeline cần sub-second response. Exact search chỉ dùng được với small datasets (<10K rows).

---

## Câu 3

**When creating an HNSW index, what does the `m` parameter control?**

- The maximum number of connections per node in the graph ✅
- The number of candidate neighbors considered during index construction
- The number of lists to partition vectors into

**Đáp án: Maximum connections per node**

HNSW là graph-based structure. `m` xác định mỗi node (vector) có tối đa bao nhiêu connections đến nearest neighbors:

```sql
CREATE INDEX ... USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
-- m = 16: mỗi vector kết nối với 16 neighbors
```

- Default: **16** — works well cho hầu hết use cases
- Higher m (32-64): better recall, tốn memory và build time hơn

Tại sao các đáp án còn lại sai:
- **Candidate neighbors during construction:** Đó là `ef_construction` — controls search width lúc build, ảnh hưởng đến index quality và build time.
- **Number of lists:** Đó là `lists` parameter của **IVFFlat**, không phải HNSW. IVFFlat partitions vectors vào lists (clusters). HNSW không dùng lists — đó là khác biệt cơ bản giữa hai algorithms.

---

## Câu 4

**You need to update embeddings for 50,000 product descriptions after switching to a new embedding model. What approach minimizes the impact on concurrent searches?**

- Batch the updates into transactions of 1,000-5,000 rows each ✅
- Update all 50,000 rows in a single transaction
- Drop the existing vector index before updating

**Đáp án: Batch updates 1,000-5,000 rows**

**Parallel column strategy** là tối ưu nhất, nhưng trong các đáp án này, **batch updates** minimize impact:
- Mỗi batch nhỏ = lock ít rows hơn = ít block concurrent reads
- Transactions hoàn thành nhanh hơn = locks được release sớm hơn
- Nếu có lỗi, chỉ mất batch hiện tại không phải toàn bộ migration

Tại sao các đáp án còn lại sai:
- **Single transaction 50,000 rows:** Transaction kéo dài rất lâu, lock tất cả 50,000 rows trong suốt quá trình, block concurrent searches hoàn toàn. Nếu có lỗi giữa chừng, rollback toàn bộ. Worst impact on concurrent searches.
- **Drop index before updating:** Dropping index không giúp gì — queries phải fall back to sequential scans trong suốt thời gian không có index, tệ hơn nhiều so với batch updates. Index cần thời gian rebuild sau đó. Tổng impact lên concurrent searches = rất cao.

---

## Câu 5

**In a hybrid search combining vector similarity with full-text search, what technique helps balance the relevance scores from both search methods?**

- Using Reciprocal Rank Fusion (RRF) to combine rankings ✅
- Multiplying the vector distance by the text relevance score
- Always returning vector search results first

**Đáp án: Reciprocal Rank Fusion (RRF)**

RRF merge rankings từ nhiều scoring methods một cách cân bằng và effective. Nó không phụ thuộc vào việc normalize scores từ các methods khác nhau — chỉ dùng rank position.

```sql
-- Weighted score combination (manual version của RRF concept)
SELECT
    id, title,
    (1 - (embedding <=> $1)) * 0.7 +
        ts_rank(to_tsvector('english', content), plainto_tsquery('english', $2)) * 0.3
        AS hybrid_score
FROM documents
WHERE to_tsvector('english', content) @@ plainto_tsquery('english', $2)
   OR embedding <=> $1 < 0.5
ORDER BY hybrid_score DESC LIMIT 10;
```

RRF là well-established technique trong information retrieval cho combining multiple ranking signals.

Tại sao các đáp án còn lại sai:
- **Multiplying scores:** Nhân vector distance (0-2, smaller=better) với text rank (0-1, larger=better) = không meaningful. Một là distance (smaller=better), một là similarity (larger=better) — nhân chúng không cho kết quả có ý nghĩa. Cần convert về cùng scale trước.
- **Always return vector first:** Không "balance" gì cả — chỉ prioritize vector và ignore full-text unless ties. Defeats the purpose của hybrid search.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Distance operator cho normalized embeddings | `<=>` cosine distance |
| 2 | Index type cho 5M embeddings, occasional batch updates | IVFFlat với appropriate lists |
| 3 | HNSW `m` parameter | Maximum connections per node |
| 4 | Update 50K embeddings, minimize impact | Batch 1,000-5,000 rows |
| 5 | Balance vector + full-text scores | RRF (Reciprocal Rank Fusion) |

---

*Module hoàn thành. PostgreSQL Learning Path tiếp theo: Module 3*
