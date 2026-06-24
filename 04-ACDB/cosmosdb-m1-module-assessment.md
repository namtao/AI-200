# Module Assessment — Build Queries for Azure Cosmos DB for NoSQL

> Khoá: AI-200 · Cosmos DB — Build queries for Azure Cosmos DB for NoSQL

---

## Câu 1

**A developer is designing a container for an AI application that stores user interaction logs. Each document includes a userId property. The application frequently retrieves all logs for a specific user. Which partition key selection provides the best performance for this access pattern?**

- Use `userId` as the partition key ✅
- Use a timestamp property as the partition key
- Use a boolean `isProcessed` property as the partition key

**Đáp án: `userId`**

Ba tiêu chí partition key tốt: (1) tồn tại trong mọi document, (2) high cardinality, (3) match access pattern.

`userId` đáp ứng cả ba: có trong mọi log, nhiều distinct values (mỗi user là một value), và app thường query "lấy logs của user X" — tức là filter theo userId.

Khi query `WHERE userId = "user-123"`, Cosmos DB route thẳng đến partition chứa logs của user đó → single-partition query → thấp RU, thấp latency.

Tại sao các đáp án còn lại sai:
- **Timestamp:** Time-based key tạo hot partition — tất cả write trong một khoảng thời gian đổ vào cùng partition. Trong peak period, partition đó bị overwhelm. Ngoài ra, query "logs của user X" sẽ cần cross-partition scan vì logs của một user trải đều across nhiều timestamp partition.
- **`isProcessed` (boolean):** Chỉ có 2 distinct values (`true`/`false`) → chỉ 2 partition → scalability bị giới hạn. Đây là ví dụ điển hình của "hot partition" với low cardinality.

---

## Câu 2

**An AI application caches model inference results in Azure Cosmos DB. The application periodically recomputes results and needs to store them regardless of whether a cached entry already exists. Which SDK method handles this requirement most effectively?**

- Use `create_item()` to insert the item
- Use `replace_item()` to update the item
- Use `upsert_item()` to insert or replace the item ✅

**Đáp án: `upsert_item()`**

`upsert_item()` làm đúng một điều: **insert nếu không tồn tại, replace nếu đã tồn tại** — không cần check trước, không cần handle exception.

```python
# Recompute inference result và cache lại
new_result = {
    "id": f"inference-{request_id}",
    "modelId": "gpt-4",
    "result": computed_result,
    "cachedAt": timestamp
}
container.upsert_item(body=new_result)    # Works regardless of prior existence
```

Tại sao các đáp án còn lại sai:
- **`create_item()`:** Fail với `CosmosResourceExistsError` (HTTP 409) nếu cached entry đã có. Cần try/except và fallback logic — phức tạp hơn không cần thiết.
- **`replace_item()`:** Fail nếu item **không tồn tại** — ngược lại với `create_item()`. Lần đầu cache chưa có gì → `replace_item()` fail. Cần check existence trước → 2 round trips thay vì 1.

---

## Câu 3

**An AI application stores product recommendations with document IDs in the format `product-{id}`. The application needs to retrieve a specific recommendation by its known ID and category. Which method provides the most efficient retrieval?**

- Use `query_items()` with a WHERE clause filtering by ID
- Use `read_item()` with the item ID and partition key ✅
- Use `query_items()` with `enable_cross_partition_query=True`

**Đáp án: `read_item()` với item ID và partition key**

Khi biết cả id và partition key, **point read** là lựa chọn duy nhất cần xem xét:

```python
# ~1 RU, route trực tiếp, không cần query parsing
item = container.read_item(
    item="product-12345",
    partition_key="electronics"
)
```

Point read cost ~1 RU và route trực tiếp đến đúng partition — không có query parsing overhead, không scan item nào khác.

Tại sao các đáp án còn lại sai:
- **`query_items()` với WHERE ID:** Query `WHERE p.id = "product-12345"` tốn RU hơn point read (3-5 RU vs 1 RU) vì cần query engine parsing. Nếu có cả partition key, Cosmos DB có thể optimize, nhưng vẫn không efficient bằng point read.
- **`query_items()` với `enable_cross_partition_query=True`:** Worst option — cross-partition query fan out to ALL partitions để find một item. Có thể tốn hàng chục RU cho một item duy nhất. Hoàn toàn ngược với optimization.

---

## Câu 4

**A developer is building a search feature that accepts user-provided filter values. The feature filters products by category and maximum price. What is the primary reason to use parameterized queries instead of string concatenation?**

- Parameterized queries prevent injection attacks and enable query plan caching ✅
- Parameterized queries automatically convert data types
- Parameterized queries run faster than queries with literal values

**Đáp án: Prevent injection attacks + query plan caching**

```python
# Parameterized — safe và cached
query = "SELECT * FROM p WHERE p.categoryId = @category AND p.price < @maxPrice"
parameters = [
    {"name": "@category", "value": user_input_category},   # User input
    {"name": "@maxPrice", "value": user_input_price}
]

# String concatenation — NGUY HIỂM
query = f"SELECT * FROM p WHERE p.categoryId = '{user_input}'"
# user_input = "'; DROP CONTAINER products--" → injection!
```

Hai lợi ích chính:
1. **Security:** Parameter values không thể modify query structure — injection không có chỗ để exploit
2. **Performance:** Cosmos DB cache query plan cho cùng query template với different parameter values — repeated queries với different values dùng lại plan

Tại sao các đáp án còn lại sai:
- **Auto data type conversion:** Không phải đặc tính của parameterized queries — type conversion vẫn cần explicit, parameterization không auto-cast.
- **Chạy nhanh hơn literal values:** Parameterized queries không tự nhiên nhanh hơn về execution speed. Lợi ích performance là từ **query plan caching** khi cùng template được gọi nhiều lần, không phải mỗi individual query.

---

## Câu 5

**A data analyst notices that queries filtering products by price range consume more RUs than expected. The container uses `categoryId` as the partition key. Which optimization would most effectively reduce RU consumption?**

- Remove the ORDER BY clause from the query
- Increase the container's provisioned throughput
- Add the partition key (`categoryId`) to the WHERE clause to enable single-partition routing ✅

**Đáp án: Add partition key vào WHERE clause**

Vấn đề: Query filter theo price range nhưng không filter theo `categoryId` (partition key) → **cross-partition query** → fan out to all partitions → tốn nhiều RU.

```python
# Trước — cross-partition query, tốn RU
query = "SELECT * FROM p WHERE p.price BETWEEN @min AND @max"
items = container.query_items(query, parameters=..., enable_cross_partition_query=True)

# Sau — single-partition query, rẻ hơn
query = "SELECT * FROM p WHERE p.categoryId = @cat AND p.price BETWEEN @min AND @max"
items = container.query_items(query, parameters=..., partition_key="electronics")
```

Khi include partition key, Cosmos DB chỉ query **một partition** → giảm đáng kể RU consumption.

Tại sao các đáp án còn lại sai:
- **Remove ORDER BY:** ORDER BY thêm overhead nhưng không phải nguyên nhân chính của high RU khi query cross-partition. Removing ORDER BY giúp đôi chút nhưng không address root cause.
- **Tăng provisioned throughput:** Tăng RU/s provisioned không giảm RU consumption của mỗi query — chỉ tăng capacity. Query vẫn tốn cùng số RU, bạn chỉ có nhiều RU hơn để spend. Expensive và không fix được vấn đề inefficiency.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Chọn partition key tốt cho user interaction logs | `userId` (high cardinality, match query pattern) |
| 2 | Cache inference result — insert hoặc replace | `upsert_item()` |
| 3 | Retrieve item khi biết id + partition key | `read_item()` (point read ~1 RU) |
| 4 | Lý do dùng parameterized query | Security (injection) + query plan caching |
| 5 | Giảm RU cho query filter theo price | Add partition key vào WHERE (single-partition routing) |

---

*Module hoàn thành. Cosmos DB Learning Path tiếp theo: Module 2*
