# Bài 3 — Combine Vector Similarity Results with Metadata Filtering

> Khoá: AI-200 · Cosmos DB — Implement vector search on Azure Cosmos DB for NoSQL

---

## Tại sao cần kết hợp?

Pure semantic search không đủ cho nhiều AI application thực tế:
- User cần kết quả trong **specific category** hoặc **date range**
- User search cho **error code cụ thể** (exact match) + semantic understanding
- System cần filter theo **access permissions**

Cosmos DB cho phép kết hợp vector search với traditional filters **trong một query duy nhất**.

---

## Vector Search + Metadata Filter (WHERE)

```python
query = """
    SELECT TOP 10
        c.id,
        c.title,
        c.category,
        c.createdDate,
        VectorDistance(c.embedding, @queryVector) AS SimilarityScore
    FROM c
    WHERE c.category = @category
        AND c.createdDate > @startDate
    ORDER BY VectorDistance(c.embedding, @queryVector)
"""

results = container.query_items(
    query=query,
    parameters=[
        {"name": "@queryVector", "value": query_embedding},
        {"name": "@category", "value": "networking"},
        {"name": "@startDate", "value": "2024-01-01T00:00:00Z"}
    ],
    enable_cross_partition_query=True
)
```

Query trả về document về "networking" sau Jan 2024, ranked theo semantic similarity.

### Common filter patterns

```sql
-- Document type
WHERE c.documentType = 'troubleshooting-guide'

-- Date range
WHERE c.createdDate >= '2024-01-01' AND c.createdDate < '2025-01-01'

-- Multiple product IDs
WHERE c.productId IN ('router-x100', 'router-x200')

-- Access permissions
WHERE ARRAY_CONTAINS(c.accessGroups, @userGroup)

-- Status
WHERE c.status = 'published'
```

---

## Pre-filtering vs Post-filtering

Query optimizer tự quyết định dựa trên filter selectivity:

**Pre-filtering** — filter trước khi so sánh vector. Filter selective (loại bỏ nhiều document) → chỉ compare vector của subset nhỏ → tiết kiệm RU.

**Post-filtering** — rank tất cả theo vector, rồi mới apply filter. Preserve ranking quality nhưng có thể trả về ít kết quả hơn TOP N requested (vì một số top-ranked documents bị filter out).

Đảm bảo filter properties có **index** trong indexing policy để optimizer có thể estimate selectivity.

---

## Partition Key Optimization

Khi WHERE clause include partition key → single-partition routing:

```python
query = """
    SELECT TOP 10
        c.id,
        c.title,
        VectorDistance(c.embedding, @queryVector) AS SimilarityScore
    FROM c
    WHERE c.category = @category
    ORDER BY VectorDistance(c.embedding, @queryVector)
"""

# partition_key parameter → explicit single-partition routing
results = container.query_items(
    query=query,
    parameters=[
        {"name": "@queryVector", "value": query_embedding},
        {"name": "@category", "value": "networking"}
    ],
    partition_key="networking"    # Route đến partition "networking"
)
```

`partition_key` parameter trong `query_items()` + WHERE clause chứa partition key = single-partition vector search = ít RU, ít latency.

---

## Hybrid Search — Vector + Full-Text (RRF)

**Khi nào cần hybrid:** User search "error code 0x80070005 access denied":
- Error code → cần exact match (full-text)
- "access denied" → semantic understanding (vector)

**Reciprocal Rank Fusion (RRF)** merge rankings từ nhiều scoring functions:

```sql
SELECT TOP 10 *
FROM c
ORDER BY RANK RRF(
    VectorDistance(c.embedding, @queryVector),
    FullTextScore(c.content, @searchTerm1, @searchTerm2)
)
```

Document rank cao trong cả vector lẫn keyword → top kết quả. Document chỉ tốt ở một → vẫn xuất hiện nhưng rank thấp hơn.

### Cấu hình Full-Text Policy (cần set khi tạo container)

```json
// Full-text policy
{
    "defaultLanguage": "en-US",
    "fullTextPaths": [
        {"path": "/content", "language": "en-US"}
    ]
}

// Indexing policy — thêm fullTextIndexes
{
    "vectorIndexes": [{"path": "/embedding", "type": "diskANN"}],
    "fullTextIndexes": [{"path": "/content"}]
}
```

---

## RRF Weights — Điều chỉnh tỷ trọng

```sql
SELECT TOP 10 *
FROM c
ORDER BY RANK RRF(
    VectorDistance(c.embedding, @queryVector),
    FullTextScore(c.content, @searchTerm1, @searchTerm2),
    [2, 1]    -- vector weight=2, full-text weight=1
)
```

| Weight | Effect |
|---|---|
| Higher vector weight | Ưu tiên semantic understanding — user describe problems naturally |
| Higher full-text weight | Ưu tiên exact keyword match — user search specific terms/codes |
| Equal [1, 1] | Balanced — starting point tốt |

---

## Multi-Vector Search

Khi document có nhiều embedding (title + content):

```sql
SELECT TOP 10 *
FROM c
ORDER BY RANK RRF(
    VectorDistance(c.titleEmbedding, @queryVector),
    VectorDistance(c.contentEmbedding, @queryVector)
)
```

Document match ở cả title lẫn content → rank cao nhất.

---

## Performance Trade-offs

| Query type | RU cost | Dùng khi |
|---|---|---|
| Pure vector search | Thấp | Semantic query thuần |
| Vector + selective filter | Tương đương hoặc thấp hơn | Filter loại bỏ nhiều document (pre-filter) |
| Vector + non-selective filter | Cao hơn | Filter match hầu hết document (ít benefit) |
| Hybrid (RRF) | Cao hơn | Cần cả semantic + exact keyword |

**Test với production data volumes** — query performance thay đổi đáng kể giữa 1,000 và 1,000,000 documents.

---

## Bản chất bài này là gì?

**Một câu:** Hybrid search dùng RRF để merge kết quả vector và keyword trong 1 query — tốt nhất cho câu truy vấn có cả exact term (error code) lẫn semantic meaning (mô tả vấn đề).

### Pure Vector vs Hybrid RRF

| | Pure Vector | Vector + WHERE filter | Hybrid RRF |
|---|---|---|---|
| "WiFi slow" | ✅ Tốt | ✅ Tốt | ✅ Tốt |
| "error 0x80070005" | ❌ Kém (số không có semantic) | ❌ Kém | ✅ Tốt (FullTextScore bắt exact) |
| "networking after 2024" | ❌ Không filter date | ✅ WHERE clause | ✅ |
| RU cost | Thấp nhất | Tương đương hoặc thấp hơn | Cao nhất |
| Setup | Đơn giản | Đơn giản | Cần vector policy + full-text policy |

**WHERE filter ≠ Hybrid:** WHERE filter loại bỏ document không match trước/sau vector search. Hybrid RRF kết hợp 2 scoring functions → document tốt ở cả hai thang điểm → rank cao nhất. Khác nhau về mục đích.

**RRF weights tuning:** `[2, 1]` = vector quan trọng gấp đôi keyword. Khi user gõ natural language ("how do I fix...") → tăng vector weight. Khi user gõ specific code/term → tăng full-text weight.

---

## Checklist ghi nhớ cho AI-200

- [ ] WHERE clause kết hợp với VectorDistance trong cùng query
- [ ] **Pre-filter** (selective) = tiết kiệm RU · **Post-filter** = preserve ranking nhưng có thể ít kết quả hơn
- [ ] Include partition key trong WHERE + `partition_key=` parameter → single-partition vector search
- [ ] **Hybrid search** = `ORDER BY RANK RRF(VectorDistance(...), FullTextScore(...))`
- [ ] RRF cần cả **vector policy** và **full-text policy** được config khi tạo container
- [ ] RRF weights: `[2, 1]` = vector 2x hơn full-text
- [ ] Multi-vector: `RANK RRF(VectorDistance(c.titleEmb, @q), VectorDistance(c.contentEmb, @q))`
- [ ] Index filter properties để optimizer estimate selectivity
- [ ] Hybrid search tốn RU hơn pure vector — dùng khi thực sự cần

---

*Bài tiếp theo: Bài 4 — Use the change feed to trigger embedding refresh*
