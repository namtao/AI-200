# Bài 1 — Understand Indexes in Azure Cosmos DB

> Khoá: AI-200 · Cosmos DB — Optimize query performance for Azure Cosmos DB for NoSQL

---

## Automatic Indexing — Default Behavior

Khi tạo container, **mặc định Cosmos DB index tất cả properties** của mọi item bằng range indexes. Điều này có nghĩa query trên bất kỳ property nào cũng có thể dùng index ngay mà không cần cấu hình.

**Trade-off khi scale:**
- Mỗi indexed property tốn storage và tăng overhead khi write
- AI application lưu embedding array (1,536 numbers) bị index không cần thiết → tăng storage cost đáng kể
- Cần customize indexing policy để tối ưu

---

## 4 Index Types chính

| Index type | Hỗ trợ | Dùng khi |
|---|---|---|
| **Range** | `=`, `>`, `<`, `>=`, `<=`, `!=`, `ORDER BY`, string functions | Filter theo type, status, date, số |
| **Composite** | Multi-property `ORDER BY`, filter + sort kết hợp | Sort theo nhiều field, filter + sort khác nhau |
| **Spatial** | `ST_DISTANCE`, `ST_WITHIN`, `ST_INTERSECTS` | Location-based filtering |
| **Vector** | `VectorDistance()` | Semantic similarity search |

*(Còn có Full-text index và Tuple index — covered sau)*

---

## Indexing Modes

| Mode | Behavior | Dùng khi |
|---|---|---|
| **Consistent** (default) | Index update đồng bộ khi write | Hầu hết workload — queries phản ánh latest writes |
| **None** | Không index, chỉ point read | Specialized: bulk write, không cần query |

> **Lazy mode đã deprecated** — không dùng cho container mới, migrate về Consistent.

---

## Index Storage Costs

Các yếu tố ảnh hưởng đến index storage:
- Số lượng indexed properties
- Cardinality (nhiều distinct values = index structure lớn hơn)
- **Array elements — mỗi element được index riêng** → array lớn = index lớn

**Với AI application:** Embedding array (1,536 numbers) trong range index = storage tốn rất nhiều mà không có ích. Luôn **exclude embedding path** khỏi range index.

---

## Path Syntax

```
/*                  → Tất cả properties, recursive
/propertyName/?     → Scalar value của property cụ thể
/arrayName/[]       → Tất cả elements trong array
/nested/path/*      → Tất cả properties dưới nested path
```

**Precedence:** Path cụ thể hơn thắng. Có thể exclude `/*` rồi include selective paths.

---

## Default vs Selective Indexing Policy

### Default — index tất cả

```json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [{ "path": "/*" }],
  "excludedPaths": [{ "path": "/\"_etag\"/?" }]
}
```

### Selective — chỉ index properties được query

```json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [
    { "path": "/title/?" },
    { "path": "/category/?" },
    { "path": "/createdDate/?" },
    { "path": "/metadata/*" }
  ],
  "excludedPaths": [
    { "path": "/*" }
  ]
}
```

Exclude `/*` mặc định, rồi include cụ thể những gì cần → không index large text content, embedding array.

---

## System Properties

| Property | Indexed? | Ghi chú |
|---|---|---|
| `id` | ✅ Always | Không thể disable |
| `_ts` | ✅ Always | Không thể disable |
| `_etag` | ❌ Excluded by default | Include nếu cần filter |

---

## Partition Key và Indexing

> **Quan trọng:** Partition key **không được tự động index** khi dùng selective indexing (exclude `/*`).

Nếu queries filter theo partition key property, cần **explicitly include** trong indexing policy:

```json
"includedPaths": [
    { "path": "/tenantId/?" },    // Partition key — phải include manually
    { "path": "/status/?" }
]
```

Thiếu partition key trong index → queries filter theo partition key phải scan toàn partition.

---

## Bản chất bài này là gì?

**Một câu:** Cosmos DB tự động index tất cả (khác SQL phải tạo index thủ công), nhưng khi có embedding arrays thì mặc định sẽ lãng phí storage lớn — phải customize indexing policy để exclude embedding và include chỉ những gì query cần.

### SQL Index vs Cosmos DB Index

| | SQL Server | Cosmos DB |
|---|---|---|
| Mặc định | Không index (trừ PK) | **Index tất cả properties** |
| Tạo index | Explicit `CREATE INDEX` | Automatic (nhưng configurable) |
| Xóa index | Explicit `DROP INDEX` | Modify indexing policy |
| Trả phí khi nào | Extra storage + slower writes | Tương tự: write overhead + storage |
| Index array | Phức tạp (XML/JSON column) | Native, mỗi element index riêng |
| Index lớn nhất | Phụ thuộc type | Vector index cho embedding arrays |

**Default index-all là OK khi dataset nhỏ, problematic khi có embeddings:** 1,536 numbers × mỗi element index riêng × số documents = storage phình to nhanh. Selective indexing giải quyết bằng `excludedPaths: [{"path": "/*"}]` rồi include cụ thể.

**Partition key không tự index khi dùng selective:** Đây là trap quan trọng. Khi dùng selective policy (exclude `/*`), partition key path phải được explicitly include, nếu không queries filter theo partition key sẽ scan toàn partition.

---

## Checklist ghi nhớ cho AI-200

- [ ] Default: **tất cả properties được index** với range index
- [ ] 4 index types: Range, Composite, Spatial, **Vector**
- [ ] **Consistent** (default) = sync update khi write · **None** = disable indexing
- [ ] Lazy mode **deprecated** → migrate về Consistent
- [ ] Array element: mỗi element index riêng → large arrays = expensive
- [ ] **Exclude embedding path** khỏi range index — dùng vector index thay thế
- [ ] Precedence: path cụ thể hơn thắng
- [ ] `id` và `_ts` **luôn được index**, không disable được
- [ ] Partition key **không tự động index** khi dùng selective strategy → must include explicitly

---

*Bài tiếp theo: Bài 2 — Configure range and composite indexes*
