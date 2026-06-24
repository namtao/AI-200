# Bài 2 — Execute Vector Similarity Queries for Semantic Search

> Khoá: AI-200 · Cosmos DB — Implement vector search on Azure Cosmos DB for NoSQL

---

## VectorDistance Function

`VectorDistance` là core function cho vector search trong Cosmos DB NoSQL:

```sql
VectorDistance(c.embedding, @queryVector)
```

Parameters:
- `vector_expr_1` — path đến embedding trong document (vd. `c.embedding`)
- `vector_expr_2` — query vector (array of numbers, thường là `@queryVector` parameter)
- `bool_expr` *(optional)* — `true` = force brute-force search
- `obj_expr` *(optional)* — custom distanceFunction, dataType

**Với cosine:** Score cao hơn = similar hơn. Score 0.9 > 0.5.

---

## Basic Vector Search Query

```sql
SELECT TOP 10
    c.id,
    c.title,
    c.category,
    VectorDistance(c.embedding, @queryVector) AS SimilarityScore
FROM c
ORDER BY VectorDistance(c.embedding, @queryVector)
```

**Hai điều bắt buộc:**
1. `ORDER BY VectorDistance(...)` — sắp xếp theo relevance, kết quả similar nhất lên đầu
2. `TOP N` — giới hạn kết quả. **Không có TOP N → query cố return tất cả document → tốn RU và latency cao**

---

## Workflow: Query Text → Embedding → Vector Search

```python
from openai import AzureOpenAI

query_text = "How do I fix WiFi connection problems?"

# Bước 1: Generate query embedding — CÙNG MODEL với document embeddings
response = openai_client.embeddings.create(
    input=query_text,
    model="text-embedding-ada-002"   # Phải match model dùng khi insert documents
)
query_embedding = response.data[0].embedding

# Bước 2: Execute vector search
query = """
    SELECT TOP 10
        c.id,
        c.title,
        c.category,
        VectorDistance(c.embedding, @queryVector) AS SimilarityScore
    FROM c
    ORDER BY VectorDistance(c.embedding, @queryVector)
"""

results = container.query_items(
    query=query,
    parameters=[{"name": "@queryVector", "value": query_embedding}],
    enable_cross_partition_query=True
)

for item in results:
    print(f"{item['title']} - Score: {item['SimilarityScore']:.4f}")
```

**Quan trọng:** Query text phải được embed bằng **cùng model** với documents. Khác model = incompatible vector space = kết quả vô nghĩa.

---

## Parameterized Query cho Vector Search

Dùng `@queryVector` parameter thay vì embed vector trực tiếp vào query string:

```python
parameters = [
    {"name": "@queryVector", "value": query_embedding}   # 1,536-element array
]
```

**Lý do:** 1,536 floats inline trong query string = unwieldy string khó debug. Parameter cũng enable query plan caching.

---

## Similarity Score và Threshold

Với **cosine similarity:**

| Score | Ý nghĩa |
|---|---|
| +1.0 | Identical vectors |
| 0.7–0.9 | Highly similar — thường relevant |
| 0.5–0.7 | Moderately similar |
| 0.1–0.5 | Low similarity |
| 0.0 hoặc negative | Dissimilar |

### Filter theo threshold

```sql
SELECT TOP 10
    c.id,
    c.title,
    VectorDistance(c.embedding, @queryVector) AS SimilarityScore
FROM c
WHERE VectorDistance(c.embedding, @queryVector) > 0.7
ORDER BY VectorDistance(c.embedding, @queryVector)
```

Threshold 0.7 = highly relevant results. Bắt đầu ở 0.7, điều chỉnh dựa trên test với representative queries.

---

## TOP N — Chọn số lượng phù hợp

| Use case | TOP N | Lý do |
|---|---|---|
| RAG application | 5–10 | LLM context không cần quá nhiều, token cost tăng |
| User-facing search | 10–20 | User duyệt qua, có thể paginate |
| Recommendation | 3–5 | Không muốn clutter UI |

**Ít results = query performance tốt hơn** — ít data processed và transferred.

---

## Indexed Search vs Brute-Force

**Default (indexed):** Dùng configured vector index (DiskANN) → fast approximate results.

```sql
-- Default: dùng index
VectorDistance(c.embedding, @queryVector)
```

**Brute-force:** Force so sánh với mọi document → exact results nhưng chậm và tốn RU.

```sql
-- Brute-force: third parameter = true
VectorDistance(c.embedding, @queryVector, true)
```

**Dùng brute-force khi:** Evaluation/testing, small dataset (< 1,000 vectors), hoặc khi cần 100% exact accuracy.

**Tránh brute-force cho production large dataset:** Scale poorly, tốn RU, latency cao.

---

## Query Performance Optimization

```python
# Chỉ project fields cần thiết, không dùng SELECT *
query = """
    SELECT TOP 10
        c.id,
        c.title,
        VectorDistance(c.embedding, @queryVector) AS SimilarityScore
    FROM c
    ORDER BY VectorDistance(c.embedding, @queryVector)
"""
```

**Các yếu tố ảnh hưởng performance:**
- **Index type:** DiskANN tốt nhất cho large dataset (>50K vectors)
- **TOP N:** Càng nhỏ càng tốt
- **Partition key:** Include để route single-partition
- **Projection:** Chỉ lấy fields cần thiết
- **Monitor RU:** `x-ms-request-charge` header

---

## Bản chất bài này là gì?

**Một câu:** Vector similarity query = "tìm document có ý nghĩa gần nhất với query" — dùng `VectorDistance` + `ORDER BY` + `TOP N`, và query text phải được embed bằng cùng model với documents.

### Keyword Search vs Semantic Search

| | Keyword (CONTAINS, LIKE) | Semantic (VectorDistance) |
|---|---|---|
| Match khi | Exact substring xuất hiện | Ý nghĩa tương đồng |
| "WiFi" vs "wireless" | ❌ Không match (khác từ) | ✅ Match (cùng concept) |
| Error code "0x800" | ✅ Exact match | ❌ Không có semantic |
| Cần cùng model | Không | ✅ BẮT BUỘC |
| Cost | Thấp (range index) | Cao hơn (vector index) |
| Dùng khi | Exact term, code, ID | Natural language, description |

**`TOP N` và `ORDER BY` là bắt buộc không phải convention:** Thiếu TOP N → query cố return tất cả vector distances trong container → RU có thể cực lớn. Thiếu ORDER BY → kết quả không có nghĩa gì.

**Brute-force = `true` là tham số thứ 3:** `VectorDistance(c.embedding, @q, true)` → so sánh với tất cả document, không dùng index. Dùng khi evaluation/testing hoặc dataset nhỏ, không dùng production.

---

## Checklist ghi nhớ cho AI-200

- [ ] `VectorDistance(c.embedding, @queryVector)` — core function
- [ ] **Luôn có `ORDER BY VectorDistance(...)` và `TOP N`** trong vector search
- [ ] Query embedding phải dùng **cùng model** với document embeddings
- [ ] Dùng `@queryVector` parameter, không embed vector trực tiếp vào query
- [ ] Cosine score: 0.7+ = highly similar, 0.5-0.7 = moderate, < 0.5 = low
- [ ] WHERE với threshold: `VectorDistance(...) > 0.7`
- [ ] Brute-force: third parameter `true` → exact nhưng slow, tránh cho production
- [ ] RAG: TOP 5-10 · User search: TOP 10-20 · Recommendation: TOP 3-5
- [ ] `enable_cross_partition_query=True` khi query across all partitions

---

*Bài tiếp theo: Bài 3 — Combine vector similarity results with metadata filtering*
