# Module Assessment — Implement Vector Storage in Azure Managed Redis

> Khoá: AI-200 · Redis — Implement vector storage in Azure Managed Redis

---

## Câu 1

**Which distance metric should you use for text embeddings?**

- COSINE ✅
- L2 (Euclidean)
- IP (Inner Product)

**Đáp án: COSINE**

Text embedding models (OpenAI, Cohere, Sentence Transformers) được trained để represent semantic meaning thông qua **angle** giữa vectors. COSINE đo chính xác điều này — ignoring magnitude, chỉ tính direction (semantic meaning).

L2 đo straight-line distance bao gồm cả magnitude — không phù hợp với text embeddings vì word frequency hay document length (magnitude) không nên ảnh hưởng đến semantic similarity.

Tại sao các đáp án còn lại sai:
- **L2:** Dùng cho image embeddings và spatial data nơi magnitude quan trọng. Với text embeddings được trained cho cosine similarity, L2 trả về mathematically valid nhưng semantically wrong results.
- **IP:** Chỉ dùng cho pre-normalized embeddings trong specialized scenarios. Không phải general choice cho text.

---

## Câu 2

**When should you choose HNSW indexing over FLAT indexing for vector search?**

- When you have large datasets (over 10,000 vectors) and need fast queries with acceptable 95-99% accuracy ✅
- When you need perfect 100% accuracy for all queries
- When you have fewer than 1,000 vectors to index

**Đáp án: Large datasets, fast queries, 95-99% accuracy acceptable**

HNSW là designed cho production AI systems: approximate nearest neighbor search với multi-layer graph structure cho phép sub-10ms queries ngay cả với millions of vectors, bằng cách chỉ examine small fraction của total vectors.

**Khi nào dùng HNSW:** >10,000 vectors, production systems, latency requirement <10ms, 95-99% recall OK.

Tại sao các đáp án còn lại sai:
- **Need 100% accuracy:** Đó là FLAT — exact brute-force search comparing every vector. Với large datasets FLAT quá chậm.
- **<1,000 vectors:** Dataset nhỏ → FLAT là fine và đơn giản hơn. HNSW overhead không cần thiết ở scale nhỏ.

---

## Câu 3

**Which data type should you use for vector storage in most AI applications?**

- FLOAT32 ✅
- FLOAT64
- INT32

**Đáp án: FLOAT32**

FLOAT32 (single-precision, 4 bytes/dimension) là standard vì:
- **6 KB cho 1536-dim vector** — 50% ít hơn FLOAT64
- Sufficient precision — AI embedding models có inherent noise, extra precision không có meaningful improvement
- Compatible với tất cả major embedding models
- Better performance — smaller data = faster memory transfers

```
1M vectors × 1536 dim × 4 bytes (FLOAT32) = 6 GB
1M vectors × 1536 dim × 8 bytes (FLOAT64) = 12 GB
```

Tại sao các đáp án còn lại sai:
- **FLOAT64:** Doubles memory và slows queries. Models không produce values với 15-digit precision — extra precision là waste.
- **INT32:** Integer type không phù hợp cho floating-point embeddings. Embeddings là fractional values (0.123, -0.456...) cần floating-point representation.

---

## Câu 4

**When should you use Redis Hash instead of JSON for storing vectors?**

- When you have flat data models and need maximum memory efficiency and query performance ✅
- When your data has nested structures or multiple vectors per document
- When you need JSON query capabilities

**Đáp án: Flat data models, maximum memory efficiency**

Hash dùng **binary bytes** (`tobytes()`) để store vectors — compact, minimal overhead. Query nhanh hơn JSON. Phù hợp khi data model đơn giản: name, price, category, embedding — flat fields không có nesting.

```python
# Hash: compact binary
redis_client.hset("product:12345", mapping={
    "name": "Widget",
    "price": "29.99",
    "embedding": embedding.tobytes()    # Binary
})
```

Tại sao các đáp án còn lại sai:
- **Nested structures or multiple vectors:** Đó là khi dùng **JSON** — JSON supports nested objects và arrays naturally, Hash không support nesting.
- **JSON query capabilities:** Cũng là khi dùng JSON — `$. JSONPath syntax`, JSON operators. Hash không có JSON query capabilities.

---

## Câu 5

**What does the EF_RUNTIME parameter control in HNSW queries?**

- The tradeoff between query speed and accuracy by controlling how many graph nodes are examined ✅
- The maximum number of results returned by the query
- The distance metric used for similarity calculations

**Đáp án: Speed/accuracy tradeoff — number of graph nodes examined**

HNSW graph navigation: EF_RUNTIME = "effort level" — bao nhiêu nodes trong graph được explore.

- Thấp (10-50) → ít nodes → faster, có thể miss some good matches
- Cao (100-200) → nhiều nodes → slower, better recall

```python
# High accuracy (slower)
Query("*=>[KNN 10 @embedding $query_vec EF_RUNTIME 200 AS score]")

# Fast (less accurate)
Query("*=>[KNN 10 @embedding $query_vec EF_RUNTIME 20 AS score]")
```

Start với 50, increase đến 100-200 nếu cần recall cao hơn.

Tại sao các đáp án còn lại sai:
- **Maximum results returned:** Đó là số **K** trong `KNN K` — ví dụ `KNN 10` trả về 10 results. EF_RUNTIME không control số lượng kết quả.
- **Distance metric:** Distance metric được set khi tạo index (`DISTANCE_METRIC: COSINE`), không thể thay đổi per-query qua EF_RUNTIME.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Distance metric cho text embeddings | COSINE |
| 2 | Chọn HNSW khi nào | >10K vectors, fast queries, 95-99% accuracy OK |
| 3 | Data type cho AI embeddings | FLOAT32 |
| 4 | Hash vs JSON | Hash = flat models, max efficiency |
| 5 | EF_RUNTIME parameter | Speed/accuracy tradeoff — graph nodes examined |

---

*Module hoàn thành. Redis Learning Path kết thúc.*
