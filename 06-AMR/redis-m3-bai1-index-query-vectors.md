# Bài 1 — Index and Query Vector Data

> Khoá: AI-200 · Redis — Implement vector storage in Azure Managed Redis

---

## Tổng quan

Azure Managed Redis hỗ trợ vector database thông qua **RediSearch**, cho phép lưu và query embeddings cho semantic search, recommendation, và RAG.

---

## Tạo Vector Index

### Schema Definition

```python
from redis.commands.search.field import TextField, VectorField
from redis.commands.search.indexDefinition import IndexDefinition, IndexType

schema = (
    TextField("title"),
    TextField("content"),
    VectorField(
        "embedding",               # Field name trong Redis hash
        "HNSW",                    # Algorithm: FLAT hoặc HNSW
        {
            "TYPE": "FLOAT32",     # Standard cho AI embeddings
            "DIM": 1536,           # Phải match embedding model output
            "DISTANCE_METRIC": "COSINE"   # COSINE cho text, L2 cho image
        }
    )
)

redis_client.ft("idx:documents").create_index(
    fields=schema,
    definition=IndexDefinition(
        prefix=["doc:"],           # Index keys bắt đầu bằng "doc:"
        index_type=IndexType.HASH
    )
)
```

Khi tạo index, Redis tự động index mọi key match prefix.

### FLAT vs HNSW

| | FLAT | HNSW |
|---|---|---|
| Accuracy | 100% exact | ~95-99% approximate |
| Speed | O(n) — slow for large | Sub-10ms even millions |
| Dùng khi | <10,000 vectors, need perfect accuracy | >10,000 vectors, production |

---

## Ingest Vector Data

Vectors lưu dưới dạng **binary bytes** (`tobytes()`):

```python
import numpy as np

embedding = np.array([0.1, 0.2, 0.3, ...], dtype=np.float32)

redis_client.hset(
    "doc:001",
    mapping={
        "title": "Getting Started with Azure AI",
        "content": "Azure AI Services provide pre-built models...",
        "embedding": embedding.tobytes(),   # NumPy array → bytes
        "category": "documentation"
    }
)
```

### Bulk Ingest với Pipeline

```python
pipeline = redis_client.pipeline()

for doc in documents:
    vector_bytes = np.array(doc['embedding'], dtype=np.float32).tobytes()
    pipeline.hset(f"doc:{doc['id']}", mapping={
        "title": doc['title'],
        "content": doc['content'],
        "embedding": vector_bytes,
        "category": doc.get('category', 'general')
    })

pipeline.execute()    # Tất cả HSETs trong một round trip → 10-100x throughput
```

---

## Query — KNN Search

```python
from redis.commands.search.query import Query

query_bytes = np.array(query_embedding, dtype=np.float32).tobytes()

query = (
    Query("*=>[KNN 5 @embedding $query_vec AS score]")
    .return_fields("title", "content", "score")
    .sort_by("score")
    .dialect(2)    # Required cho vector queries
)

results = redis_client.ft("idx:documents").search(
    query,
    query_params={"query_vec": query_bytes}
)
```

Query syntax breakdown:
- `*` = search all documents
- `=>[KNN 5 @embedding $query_vec AS score]` = find 5 nearest neighbors trong field `embedding`
- `.dialect(2)` = required cho vector queries

Score: COSINE distance — **0.0 = identical, 2.0 = opposite**. Thấp hơn = similar hơn.

---

## Hybrid Search (Vector + Metadata Filter)

```python
hybrid_query = Query(
    "@category:{documentation}=>[KNN 3 @embedding $query_vec AS score]"
).return_fields("title", "category", "score").sort_by("score").dialect(2)

results = redis_client.ft("idx:documents").search(
    hybrid_query,
    query_params={"query_vec": query_bytes}
)
```

Filter trước (`@category:{documentation}`) → sau đó KNN chỉ trong filtered set.

---

## Range Query (Threshold-based)

Thay vì fixed K results, trả về tất cả vectors trong khoảng distance:

```python
range_query = Query(
    "@embedding:[VECTOR_RANGE 0.2 $query_vec]=>{$YIELD_DISTANCE_AS: score}"
).return_fields("title", "score").sort_by("score").dialect(2)
```

0.2 = distance threshold. Kết quả có thể là 3 hoặc 300 documents tùy data.

---

## EF_RUNTIME — Tune HNSW Accuracy

```python
# Higher EF_RUNTIME = better accuracy, slower queries
query = Query("*=>[KNN 10 @embedding $query_vec EF_RUNTIME 200 AS score]")
# Start with 50, increase to 100-200 nếu cần recall cao hơn
```

---

## Bản chất bài này là gì?

**Một câu:** RediSearch trong Azure Managed Redis cho phép KNN vector search với hybrid metadata filtering — setup đúng index (DIM match, metric đúng, dialect 2) là điều kiện cần thiết.

### So sánh FLAT vs HNSW

| | FLAT | HNSW |
|---|---|---|
| Accuracy | 100% exact | 95-99% approximate |
| Time complexity | O(n) linear scan | Sub-linear graph traversal |
| Query speed (1M vectors) | ~seconds | Sub-10ms |
| Memory overhead | Thấp | Cao hơn (graph structure) |
| Dùng khi | <10K vectors, dev/test | >10K vectors, production |

**DIM mismatch = silent failure hoặc error:** Embedding model output 1536 dimensions, index define 768 → ingestion fail hoặc query trả về garbage. DIM phải khớp chính xác với model.

**Exam trap — `.dialect(2)` là bắt buộc:** Quên `.dialect(2)` trong vector query → syntax error hoặc wrong results. Đây là requirement riêng của vector search, không phải optional optimization.

---

## Checklist ghi nhớ cho AI-200

- [ ] Vector index cần `VectorField` với TYPE, DIM, DISTANCE_METRIC
- [ ] DIM phải **match chính xác** embedding model output — sai = error
- [ ] FLAT = exact (100%) · HNSW = approximate (95-99%), sub-10ms
- [ ] Lưu vector: `embedding.tobytes()` (NumPy → bytes)
- [ ] Bulk ingest: pipeline → 10-100x throughput
- [ ] KNN query syntax: `*=>[KNN N @field $param AS score]`
- [ ] `.dialect(2)` bắt buộc cho vector queries
- [ ] Hybrid: filter expression **trước** `=>` operator
- [ ] VECTOR_RANGE = threshold-based retrieval thay vì fixed K
- [ ] `EF_RUNTIME` = tune HNSW accuracy/speed tradeoff
- [ ] COSINE score: **0.0 = identical, 2.0 = opposite**

---

*Bài tiếp theo: Bài 2 — Choose vector types and indexing strategies*
