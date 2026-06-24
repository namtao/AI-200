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

*Bài tiếp theo: Bài 4 — Scale for high-volume workloads + Connection optimization*
