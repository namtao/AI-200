# Bài 3 — Optimize Data Layout

> Khoá: AI-200 · PostgreSQL — Optimize vector search in Azure Database for PostgreSQL

---

## Vector Storage

Dimension phải match embedding model — sai dimension = insertion error.

| Dimensions | Bytes/vector | 1M vectors |
|---|---|---|
| 384 | ~1.5 KB | ~1.5 GB |
| 768 | ~3 KB | ~3 GB |
| **1536** | **~6 KB** | **~6 GB** |
| 3072 | ~12 KB | ~12 GB |

HNSW index overhead: +50% của vector column size. 1M × 1536-dim = ~6 GB data + ~3 GB index = ~9 GB.

---

## Structured Columns vs JSONB

| | Structured | JSONB |
|---|---|---|
| B-tree index | ✅ equality + range | ❌ GIN chỉ cho containment |
| Query performance | Fast | Slower (parse overhead) |
| Schema | Fixed | Flexible |
| Planner stats | Accurate | Less precise |

**Kết hợp cả hai — best practice:**

```sql
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    embedding vector(1536),
    -- Structured cho frequently filtered
    category_id INTEGER NOT NULL,
    price NUMERIC(10,2) NOT NULL,
    in_stock BOOLEAN DEFAULT true,
    -- JSONB cho dynamic/rarely filtered
    attributes JSONB DEFAULT '{}'
);
```

---

## Metadata Indexes

```sql
-- Single column
CREATE INDEX idx_products_category ON products (category_id);

-- Composite: leftmost column phải match filter
CREATE INDEX idx_products_category_price ON products (category_id, price);

-- Partial index: nhỏ hơn, nhanh hơn
CREATE INDEX idx_products_instock_category ON products (category_id)
WHERE in_stock = true;

-- GIN cho JSONB containment (@>, ?, ?|, ?&)
CREATE INDEX idx_products_attributes ON products USING gin (attributes);

-- Expression index cho JSONB range queries
CREATE INDEX idx_products_json_price ON products (((attributes->>'price')::numeric));
```

---

## Filter + Vector Search Patterns

### Pre-filter (efficient khi filter selective)

```sql
SELECT id, name, embedding <=> $1 AS distance
FROM products
WHERE category_id = 5
  AND in_stock = true
  AND price BETWEEN 100 AND 500
ORDER BY embedding <=> $1
LIMIT 10;
```

Filter giảm search space trước → ít vector calculations hơn.

### Post-filter (khi filter không selective)

```sql
WITH candidates AS (
    SELECT id, name, price, in_stock, embedding <=> $1 AS distance
    FROM products
    ORDER BY embedding <=> $1
    LIMIT 100    -- Fetch more than needed
)
SELECT id, name, distance
FROM candidates
WHERE in_stock = true AND price BETWEEN 100 AND 500
ORDER BY distance
LIMIT 10;
```

Dùng khi filter matches phần lớn rows → không helpful cho pre-filter.

Verify plan với `EXPLAIN ANALYZE` — check Index Scan vs Seq Scan.

---

## Table Partitioning

Phù hợp khi: tens of millions of rows, queries naturally filter by partition key.

```sql
-- Range partitioning by date
CREATE TABLE user_interactions (
    id BIGSERIAL,
    user_id BIGINT NOT NULL,
    embedding vector(768),
    created_at TIMESTAMP NOT NULL,
    interaction_type TEXT
) PARTITION BY RANGE (created_at);

CREATE TABLE user_interactions_2025_01
    PARTITION OF user_interactions
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

Index trên parent → auto-create trên tất cả partitions. Mỗi partition có index riêng → rebuild độc lập. Cross-partition queries có thể chậm hơn single table.

---

## Bản chất bài này là gì?

**Một câu:** Data layout optimization cho vector search là 3 quyết định — structured columns vs JSONB cho metadata (ảnh hưởng index efficiency), pre-filter vs post-filter pattern (ảnh hưởng search scope), và partitioning (chỉ khi tens of millions rows với natural partition key).

### So sánh với MongoDB vs Cassandra vs DynamoDB (metadata + vector layout)

| Design decision | PostgreSQL (pgvector) | MongoDB | Cassandra | DynamoDB |
|---|---|---|---|---|
| Structured metadata index | B-tree (equality + range) | Compound index | Partition + clustering key | GSI/LSI |
| JSONB/document metadata | GIN (containment only) | ✅ All field types | ❌ | Map type (no secondary index) |
| Pre-filter (reduce vector scope) | `WHERE category = X` before ORDER BY | `$match` before `$vectorSearch` | Partition key filter | Query + filter |
| Post-filter (fetch more, then filter) | `WITH candidates AS (... LIMIT 100)` | `numCandidates` parameter | N/A | Filter expression |
| Partial index | ✅ `WHERE in_stock = true` | Partial index | ❌ | ❌ |
| Range partitioning | `PARTITION BY RANGE (date)` | Time-series collection | ✅ Native | Table design |
| Composite leftmost rule | ✅ | ✅ | ✅ (partition key first) | ✅ (PK first) |

**Pre-filter vs post-filter không chỉ là SQL style — đây là correctness issue với ANN:** ANN index tìm approximate nearest neighbors trong TOÀN TABLE. Nếu dùng post-filter với LIMIT 10 từ vector search, rồi filter bỏ 9 trong 10 results → trả về 1 result mặc dù có thể 50 matching vectors ngoài kia. `LIMIT 100` trong post-filter là cách workaround (fetch 100, expect filter keeps 10) nhưng không đảm bảo. MongoDB giải quyết bằng `numCandidates` parameter — pgvector không có native solution cho vấn đề này.

**Partial index là feature ít dùng nhất nhưng có lợi nhất cho AI app:** `CREATE INDEX ... WHERE in_stock = true` tạo index chỉ cho subset rows → nhỏ hơn → fit trong memory tốt hơn → cache hit rate cao hơn → faster. DynamoDB và Cassandra không có partial index concept. MongoDB có partial index filter. Đây là PostgreSQL strength mà thường bị bỏ qua khi thiết kế schema.

---

## Checklist ghi nhớ cho AI-200

- [ ] 1536-dim = **~6 KB/vector**, HNSW adds ~50% overhead
- [ ] Structured columns: B-tree index, equality + range efficient
- [ ] JSONB: GIN index chỉ cho containment, không cho range
- [ ] Composite index: leftmost column phải match filter
- [ ] Partial index (`WHERE in_stock = true`) giảm size, tốt cho selective filter
- [ ] Pre-filter → ít vector calculations; Post-filter → fetch more candidates then filter
- [ ] `EXPLAIN ANALYZE` để verify Index Scan vs Seq Scan
- [ ] `ANALYZE products;` sau khi load data → planner stats accurate
- [ ] Partitioning: tens of millions rows, queries filter by partition key

---

[← Bài 2](./postgresql-m3-bai2-choose-indexes.md) · [🏠 Mục lục](../README.md) · [Bài 4 →](./postgresql-m3-bai4-scale-connections.md)
