# Module Assessment — Optimize Query Performance for Azure Cosmos DB for NoSQL

> Khoá: AI-200 · Cosmos DB — Optimize query performance for Azure Cosmos DB for NoSQL

---

## Câu 1

**A developer needs to optimize a query that filters documents by `documentType` and sorts results by `uploadDate` in descending order. Which index configuration best supports this query pattern?**

- Create a composite index with `documentType` (ascending) followed by `uploadDate` (descending) ✅
- Create separate range indexes on `documentType` and `uploadDate`
- Create a composite index with `uploadDate` (descending) followed by `documentType` (ascending)

**Đáp án: Composite index (documentType ASC, uploadDate DESC)**

Query pattern: filter on `documentType` + sort by `uploadDate DESC`. Để tối ưu bằng composite index, rewrite query include filter property trong ORDER BY:

```sql
SELECT * FROM c
WHERE c.documentType = 'pdf'
ORDER BY c.documentType, c.uploadDate DESC
```

Composite index config:
```json
[
  { "path": "/documentType", "order": "ascending" },
  { "path": "/uploadDate", "order": "descending" }
]
```

**Quy tắc thiết kế:** Equality filter property đặt trước, range/sort property đặt sau. Order phải khớp chính xác với ORDER BY clause.

Tại sao các đáp án còn lại sai:
- **Separate range indexes:** Range index riêng không tối ưu cho multi-property sort. Query `ORDER BY documentType, uploadDate DESC` cần composite index — không thể dùng 2 range indexes riêng lẻ cho multi-property ORDER BY.
- **Composite (uploadDate DESC, documentType ASC):** Thứ tự sai. Composite index phải match **chính xác** sequence của ORDER BY. `documentType` là equality filter cần đặt **trước**, `uploadDate` là sort property đặt **sau**.

---

## Câu 2

**An AI application stores 500,000 embeddings per partition and needs to perform fast similarity searches with acceptable accuracy trade-offs. Which vector index type is most appropriate?**

- flat
- diskANN ✅
- quantizedFlat

**Đáp án: diskANN**

500,000 vectors per partition = large dataset. Selection criteria:

| Index | Threshold |
|---|---|
| flat | Small dataset, max **505 dimensions** |
| quantizedFlat | ~1K–50K vectors per partition |
| **diskANN** | **>50K vectors** per partition → đây chính xác là trường hợp này |

diskANN dùng approximate nearest neighbor algorithms, cung cấp:
- Lowest latency và RU cost ở large scale
- High recall (accuracy) dù approximate
- Support đến 4,096 dimensions

"Acceptable accuracy trade-offs" trong câu hỏi xác nhận approximate search là OK.

Tại sao các đáp án còn lại sai:
- **flat:** Brute-force, max 505 dimensions — không scale tốt cho 500K vectors, và giới hạn 505 dimensions có thể không đủ cho embedding models hiện đại (ada-002 dùng 1,536).
- **quantizedFlat:** Phù hợp cho ~1K–50K vectors, không recommended cho 500K vectors — performance degradation đáng kể ở quy mô này.

---

## Câu 3

**A team discovers that embedding arrays are consuming significant storage space. The embeddings are used only for vector similarity searches. How should they modify the indexing policy to reduce storage costs?**

- Set `indexingMode` to `none` to disable all indexing on the container
- Change the embedding data type from float32 to float16 in the range index
- Exclude the embedding path from `includedPaths` and add a vector index for the embedding property ✅

**Đáp án: Exclude embedding từ range index + add vector index**

```json
{
  "indexingMode": "consistent",
  "includedPaths": [
    { "path": "/category/?" },
    { "path": "/documentType/?" },
    { "path": "/uploadDate/?" }
  ],
  "excludedPaths": [
    { "path": "/*" },
    { "path": "/embedding/*" }    // Exclude khỏi range index
  ],
  "vectorIndexes": [
    { "path": "/embedding", "type": "diskANN" }   // Vector index riêng
  ]
}
```

Embedding array (1,536 numbers) trong **range index** = lãng phí storage vì vector search dùng **vector index** chứ không dùng range index. Exclude range index + add vector index là giải pháp đúng.

Tại sao các đáp án còn lại sai:
- **Set indexingMode to none:** Disable tất cả indexing — container chỉ dùng point read, không thể query bất kỳ property nào. Quá extreme, phá vỡ mọi query khác.
- **Change float32 to float16 in range index:** float16 giảm storage của vector data, nhưng vẫn duy trì range index cho embedding — vẫn tốn storage cho range index structure. Hơn nữa, `float16` là data type trong vector policy, không phải range index. Range index embedding là vô ích dù data type nào.

---

## Câu 4

**A document search application queries by category and date range, but users report that recently uploaded documents don't appear in search results. The account uses eventual consistency by default. What change would ensure users see their own uploads immediately?**

- Change the default consistency to strong consistency for all operations
- Use session consistency and pass session tokens from write operations to subsequent reads ✅
- Increase the container's provisioned throughput to speed up replication

**Đáp án: Session consistency + session tokens**

Vấn đề: User upload document → query ngay → document chưa xuất hiện. Đây là **read-your-writes** problem. **Session consistency** giải quyết chính xác vấn đề này — trong cùng session, reads luôn reflect session's writes.

```python
# Write với session token capture
session_info = {}
def capture_token(headers, result):
    session_info["token"] = headers.get("x-ms-session-token", "")

container.create_item(body=document, response_hook=capture_token)

# Query với session token → guaranteed thấy document vừa upload
results = container.query_items(
    query="SELECT * FROM c WHERE c.category = @cat",
    parameters=[{"name": "@cat", "value": "proposals"}],
    session_token=session_info.get("token")
)
```

Session consistency cũng chỉ tốn **1×** RU (không phải 2× như Strong).

Tại sao các đáp án còn lại sai:
- **Strong consistency for all operations:** Overkill và expensive. Strong = 2× RU read cost, tăng write latency, không support multiple write regions. Câu hỏi chỉ cần user thấy **uploads của chính họ** — không cần global linearizability. Session consistency giải quyết đủ với cost thấp hơn.
- **Increase provisioned throughput:** Throughput (RU/s) ảnh hưởng đến capacity và tốc độ processing, không phải replication consistency. Tăng RU/s không fix được eventual consistency — document vẫn có thể chưa được replicate khi user query ngay sau upload.

---

## Câu 5

**Query metrics show that a frequently executed query has low index utilization and high retrieved-to-output document ratios. What does this indicate and how should the developer respond?**

- The query is performing scans instead of using indexes efficiently. Analyze the query to identify which properties need indexes and add appropriate range or composite indexes. ✅
- The container has too many indexes. Remove indexes to reduce query overhead and improve index utilization metrics.
- The query is returning too many results. Add a TOP clause to limit results and improve the retrieved-to-output ratio.

**Đáp án: Query đang scan, cần thêm index**

Hai dấu hiệu trong câu hỏi:
- **Low index utilization** → query không tìm thấy index phù hợp → fall back to full scan
- **High retrieved-to-output ratio** → fetch nhiều documents từ storage nhưng chỉ return một phần nhỏ → filter không efficient

Cả hai đều chỉ ra **missing index**. Hành động đúng:
1. Xem query pattern — WHERE và ORDER BY dùng properties nào
2. Thêm range index cho filtered properties
3. Thêm composite index nếu multi-property sort hoặc filter + sort

Tại sao các đáp án còn lại sai:
- **Quá nhiều indexes, xóa bớt:** Ngược lại với vấn đề. Low index utilization = không đủ index, không phải quá nhiều. Xóa index sẽ làm tình hình tệ hơn.
- **Add TOP clause:** TOP limit kết quả **returned**, không giải quyết inefficient filtering. High retrieved-to-output ratio nghĩa là Cosmos DB đang đọc nhiều document từ storage trước khi filter — TOP không giảm số documents được scan, chỉ reduce data transferred. Cần index để reduce documents scanned.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Composite index cho filter + sort | (documentType ASC, uploadDate DESC) |
| 2 | Vector index cho 500K vectors/partition | diskANN |
| 3 | Giảm storage từ embedding arrays | Exclude range index + add vector index |
| 4 | User không thấy document vừa upload | Session consistency + session tokens |
| 5 | Low index utilization + high retrieved/output ratio | Thêm range/composite index cho missing properties |

---

*Module hoàn thành. Cosmos DB Learning Path kết thúc.*

---

[← Bài 3](./cosmosdb-m3-bai3-vector-indexes-consistency.md) · [🏠 Mục lục](../README.md) · [Tổng hợp →](./cosmosdb-summary.md)
