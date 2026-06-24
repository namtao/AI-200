# Bài 3 — Query Azure Cosmos DB for NoSQL

> Khoá: AI-200 · Cosmos DB — Build queries for Azure Cosmos DB for NoSQL

---

## Point Read vs Query — Chọn cái nào?

| | Point Read | Query |
|---|---|---|
| Cần biết | id + partition key | Filter criteria |
| RU cost | ~1 RU (thấp nhất) | 3 RU trở lên, tùy query |
| Use case | Fetch item đã biết ID | Tìm item theo điều kiện |
| Dùng khi | Có thể — luôn prefer point read | Khi không biết ID hoặc cần nhiều item |

**Rule:** Thiết kế data model để enable point read cho access pattern phổ biến nhất.

---

## Basic SELECT Queries

```sql
-- Lấy tất cả fields
SELECT * FROM products p

-- Projection — chỉ lấy fields cần thiết
SELECT p.id, p.name, p.price, p.categoryId
FROM products p
```

**Alias:** Dùng `p` (hoặc bất kỳ chữ gì) sau FROM để reference item properties.

### Execute query (Python)

```python
query = "SELECT p.id, p.name, p.price FROM products p"

items = container.query_items(query=query, enable_cross_partition_query=True)

for item in items:
    print(f"{item['name']}: ${item['price']}")
```

---

## WHERE — Filter Results

```sql
-- Comparison operators
SELECT * FROM products p
WHERE p.categoryId = "electronics" AND p.price < 500

-- String functions
SELECT * FROM products p
WHERE CONTAINS(p.name, "Speaker") AND p.price BETWEEN 50 AND 200

-- IN operator
SELECT * FROM products p
WHERE p.categoryId IN ("electronics", "appliances", "computers")
```

**String functions phổ biến:** `CONTAINS`, `STARTSWITH`, `ENDSWITH`, `UPPER`, `LOWER`

**Note:** String comparison case-sensitive mặc định.

---

## Parameterized Queries — Luôn Dùng cho User Input

```python
query = """
    SELECT * FROM products p
    WHERE p.categoryId = @category AND p.price < @maxPrice
"""

parameters = [
    {"name": "@category", "value": "electronics"},
    {"name": "@maxPrice", "value": 500.00}
]

items = container.query_items(
    query=query,
    parameters=parameters,
    enable_cross_partition_query=True
)
```

**Hai lý do dùng parameterized query:**
1. **Security** — prevent injection attack (user input không thể modify query structure)
2. **Performance** — Cosmos DB cache query plan, reuse cho repeated query với different values

---

## Single-Partition vs Cross-Partition Query

### Single-partition (rẻ hơn) — include partition key

```python
# Method 1: specify partition_key explicitly
items = container.query_items(
    query=query,
    parameters=parameters,
    partition_key="electronics"    # Route to one partition
)

# Method 2: partition key trong WHERE clause
# Nếu WHERE có categoryId = "electronics" (partition key)
# và enable_cross_partition_query=False → route to single partition
```

### Cross-partition (đắt hơn) — fan out to all partitions

```python
items = container.query_items(
    query=query,
    parameters=parameters,
    enable_cross_partition_query=True    # Scan all partitions
)
```

**Rule:** Luôn include partition key trong query khi có thể. Cross-partition query cần thiết khi search across all data hoặc filter không bao gồm partition key.

---

## ORDER BY và Pagination

```sql
SELECT * FROM products p
WHERE p.categoryId = "electronics"
ORDER BY p.price DESC
```

```python
query_iterable = container.query_items(
    query=query,
    parameters=[{"name": "@category", "value": "electronics"}],
    partition_key="electronics",
    max_item_count=50    # Items per page
)

# Iterate through pages
for page in query_iterable.by_page():
    for item in page:
        process_item(item)
```

**max_item_count:** Balance giữa ít round trips (page lớn) và ít memory (page nhỏ).

---

## Projections và VALUE keyword

### Custom result shape

```python
query = """
    SELECT VALUE {
        "productId": p.id,
        "productName": p.name,
        "currentPrice": p.price,
        "isAvailable": p.quantity > 0,
        "category": p.categoryId
    }
    FROM products p
    WHERE p.categoryId = @category
"""
```

`VALUE` keyword trả về kết quả không bị wrap trong object. Dùng để return array of scalars:

```sql
-- Returns: ["Smart Speaker", "Wireless Headphones"]
SELECT VALUE p.name
FROM products p
WHERE p.categoryId = "electronics"
```

---

## Aggregate Functions

```sql
SELECT VALUE COUNT(1)
FROM products p
WHERE p.categoryId = "electronics"
```

```python
query = """
    SELECT
        COUNT(1) as totalProducts,
        AVG(p.price) as averagePrice,
        MIN(p.price) as minPrice,
        MAX(p.price) as maxPrice
    FROM products p
    WHERE p.categoryId = @category
"""

items = list(container.query_items(
    query=query,
    parameters=[{"name": "@category", "value": "electronics"}],
    partition_key="electronics"
))

stats = items[0]
print(f"Products: {stats['totalProducts']}, Avg: ${stats['averagePrice']:.2f}")
```

**Aggregates:** `COUNT`, `SUM`, `AVG`, `MIN`, `MAX` — scan all matching items, có thể tốn nhiều RU với large dataset. Dùng filter để limit scope.

---

## Query Arrays

```sql
-- Check array contains value
SELECT * FROM products p
WHERE ARRAY_CONTAINS(p.features, "wifi")

-- JOIN để flatten array và filter
SELECT p.name, f AS feature
FROM products p
JOIN f IN p.features
WHERE f IN ("wifi", "bluetooth")
```

---

## Monitor Query Costs

```python
query_iterable = container.query_items(
    query="SELECT * FROM products p WHERE p.categoryId = @category",
    parameters=[{"name": "@category", "value": "electronics"}],
    partition_key="electronics"
)

total_rus = 0
item_count = 0

for page in query_iterable.by_page():
    headers = container.client_connection.last_response_headers or {}
    page_rus = float(headers.get('x-ms-request-charge', 0))
    total_rus += page_rus
    for item in page:
        item_count += 1

print(f"Query: {item_count} items, {total_rus:.2f} RUs")
```

---

## Query Optimization Checklist

| Optimization | Action |
|---|---|
| Include partition key | Avoid cross-partition scan |
| Project needed fields | `SELECT p.id, p.name` thay vì `SELECT *` |
| Filter early | WHERE clause reduce result set trước aggregation |
| Limit results | Dùng `TOP` khi không cần all matches |
| Use parameterized queries | Query plan caching |
| Monitor RU | `x-ms-request-charge` per page |

---

## Bản chất bài này là gì?

**Một câu:** Cosmos DB dùng SQL dialect quen thuộc nhưng document-centric — key insight là point read (~1 RU) >> query (~3+ RU), và cross-partition query = đắt nhất → thiết kế data model để tránh cross-partition.

### So sánh với SQL và MongoDB

| Feature | SQL Server | MongoDB | Cosmos DB NoSQL |
|---|---|---|---|
| Query syntax | SQL đầy đủ | MQL (`find`, `aggregate`) | SQL dialect (subset) |
| JOIN across tables | ✅ | ✅ (`$lookup`) | ❌ Chỉ JOIN trong document |
| Array query | Không native | ✅ `$elemMatch` | `ARRAY_CONTAINS`, `JOIN f IN p.features` |
| Parameterized query | ✅ `@param` | ✅ | ✅ `@param` (same syntax) |
| Cross-collection query | ✅ | ✅ (pipeline) | ❌ Không (một container mỗi query) |
| Cost metric | Query plan/IO | Execution stats | RU/s (`x-ms-request-charge`) |

**VALUE keyword không có trong SQL:** Trả về scalar/array trực tiếp thay vì object wrapper. `SELECT VALUE p.name` → `["Speaker", "Router"]` thay vì `[{"name": "Speaker"}, {"name": "Router"}]`.

**Partition key trong query = đắt vs rẻ hàng chục lần:** Nếu WHERE clause include partition key → route đến 1 partition → rẻ. Không có → scan tất cả partition → cross-partition → tốn gấp nhiều lần RU.

---

## Checklist ghi nhớ cho AI-200

- [ ] **Point read ~1 RU** — dùng khi biết id + partition key
- [ ] Cross-partition query = đắt hơn → luôn include partition key khi có thể
- [ ] **Parameterized query** = security (prevent injection) + performance (query plan cache)
- [ ] `partition_key=` parameter → single-partition routing
- [ ] `enable_cross_partition_query=True` → scan all partitions
- [ ] `max_item_count` → control page size khi pagination
- [ ] `VALUE` keyword → unwrap result, trả về scalar/array trực tiếp
- [ ] `ARRAY_CONTAINS(p.features, "wifi")` → filter theo array element
- [ ] Aggregates: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`
- [ ] Track `x-ms-request-charge` per page để tính total query RU

---

*Module Assessment tiếp theo*
