# Bài 3 — Optimize Redis Data Structures for Vector Storage

> Khoá: AI-200 · Redis — Implement vector storage in Azure Managed Redis

---

## Hash vs JSON — Tổng quan

| Factor | Hash | JSON |
|---|---|---|
| Memory | **Lower** (binary bytes) | Higher (JSON array overhead) |
| Query performance | **Faster** | Slightly slower |
| Data complexity | Flat fields only | Nested objects supported |
| Vector storage | Binary bytes (`tobytes()`) | Numeric array (`tolist()`) |
| Best for | Simple records, max performance | Complex documents, flexibility |

---

## Redis Hash

### Store

```python
import numpy as np

embedding = np.array([0.1, 0.2, 0.3, ...], dtype=np.float32)

redis_client.hset(
    "product:12345",
    mapping={
        "name": "Wireless Mouse",
        "price": "29.99",
        "category": "electronics",
        "embedding": embedding.tobytes()    # Binary bytes
    }
)
```

### Create Index

```python
from redis.commands.search.field import TextField, NumericField, VectorField
from redis.commands.search.indexDefinition import IndexDefinition, IndexType

schema = (
    TextField("name"),
    NumericField("price"),
    TextField("category"),
    VectorField("embedding", "HNSW", {
        "TYPE": "FLOAT32", "DIM": 1536, "DISTANCE_METRIC": "COSINE"
    })
)

redis_client.ft("idx:products").create_index(
    fields=schema,
    definition=IndexDefinition(prefix=["product:"], index_type=IndexType.HASH)
)
```

---

## Redis JSON

### Store

```python
document = {
    "name": "Wireless Mouse",
    "price": 29.99,
    "category": "electronics",
    "embedding": embedding.tolist(),    # List of floats (JSON-compatible)
    "specs": {                          # Nested objects supported
        "color": "black",
        "wireless": True
    }
}

redis_client.json().set("product:12345", "$", document)
```

### Create Index

```python
schema = (
    TextField("$.name", as_name="name"),
    NumericField("$.price", as_name="price"),
    TextField("$.category", as_name="category"),
    VectorField("$.embedding", "HNSW", {
        "TYPE": "FLOAT32", "DIM": 1536, "DISTANCE_METRIC": "COSINE"
    }, as_name="embedding")
)

redis_client.ft("idx:products").create_index(
    fields=schema,
    definition=IndexDefinition(prefix=["product:"], index_type=IndexType.JSON)
)
```

JSONPath syntax: `$.fieldname`. `as_name` để reference trong query.

---

## Storage Comparison

```python
# Hash: 1536 floats × 4 bytes = 6,144 bytes
hash_vector = embedding.tobytes()     # Compact binary

# JSON: similar bytes + JSON formatting overhead
json_vector = embedding.tolist()      # Array structure
```

Với 1M products × 1536-dim: Hash tiết kiệm memory đáng kể và query nhanh hơn.

---

## Migration Hash → JSON

```python
# Read từ Hash
hash_data = redis_client.hgetall("product:12345")

# Convert và write sang JSON
document = {
    "name": hash_data["name"],
    "price": float(hash_data["price"]),
    "category": hash_data["category"],
    "embedding": np.frombuffer(hash_data["embedding"], dtype=np.float32).tolist()
}

redis_client.json().set("product:12345", "$", document)
# Delete old index, create new index với IndexType.JSON
```

---

## Decision Guide

**Chọn Hash khi:**
- Flat data model (không có nested objects)
- Cần max memory efficiency
- Cần max query speed
- Data model đơn giản, không thay đổi cấu trúc

**Chọn JSON khi:**
- Data có nested structures (images array, variant SKUs, hierarchical categories)
- Cần nhiều vectors per document
- App đã dùng JSON format
- Flexibility quan trọng hơn raw performance

---

## Bản chất bài này là gì?

**Một câu:** Hash (binary bytes, flat fields, faster) vs JSON (tolist, nested objects, flexible) — mặc định dùng Hash trừ khi data thực sự có nested structure hoặc cần nhiều vectors per document.

### So sánh Hash vs JSON cho vector storage

| | Hash | JSON |
|---|---|---|
| Vector encoding | `embedding.tobytes()` binary | `embedding.tolist()` float array |
| IndexType | `IndexType.HASH` | `IndexType.JSON` |
| Field syntax | `"fieldname"` | `"$.fieldname"` + `as_name` |
| Nested objects | ❌ | ✅ |
| Multiple vectors | ❌ | ✅ |
| Memory (vector data) | Binary — thấp hơn | Array với JSON overhead |
| Query speed | Nhanh hơn | Slightly slower |

**tobytes() vs tolist() là exam trap cổ điển:** Hash dùng `embedding.tobytes()` → binary bytes. JSON dùng `embedding.tolist()` → list of floats. Nhầm lẫn = index tạo thành công nhưng vector search fail hoàn toàn.

**JSON field syntax bắt buộc có `$.`:** `TextField("$.name", as_name="name")` — bỏ `$.` hoặc bỏ `as_name` → query không tìm được field. Hash không cần prefix này.

---

## Checklist ghi nhớ cho AI-200

- [ ] **Hash** = binary bytes (`tobytes()`), flat fields, faster, less memory
- [ ] **JSON** = numeric array (`tolist()`), nested objects, slightly slower
- [ ] Hash: `IndexType.HASH` · JSON: `IndexType.JSON`
- [ ] JSON field syntax: `$.fieldname` với `as_name` alias
- [ ] 1536-dim FLOAT32 = **6 KB/vector** (Hash và JSON tương đương về vector size)
- [ ] Hash preferred cho simple records · JSON cho complex/nested data
- [ ] Migration: `hgetall` → convert → `json().set` → delete old index, create new

---

*Module Assessment tiếp theo*
