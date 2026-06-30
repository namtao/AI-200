# Bài 2 — Configure Range and Composite Indexes

> Khoá: AI-200 · Cosmos DB — Optimize query performance for Azure Cosmos DB for NoSQL

---

## Range Indexes — Filter Operations

Range index hỗ trợ: equality (`=`), range (`>`, `<`, `>=`, `<=`, `!=`), `ORDER BY` single property, string functions (`CONTAINS`, `STARTSWITH`, `ENDSWITH`).

```sql
-- Range index on documentType và uploadDate
SELECT * FROM c
WHERE c.documentType = 'pdf'
  AND c.uploadDate > '2024-01-01'
```

Không có index → full scan toàn partition → RU tốn tỷ lệ với data size thay vì result size.

---

## Composite Indexes — Multi-property Sort và Filter

### Khi nào BẮT BUỘC có composite index?

**ORDER BY với 2+ properties:**

```sql
-- Yêu cầu composite index (relevanceScore DESC, uploadDate DESC)
SELECT * FROM c
ORDER BY c.relevanceScore DESC, c.uploadDate DESC
```

Không có composite index → **query fail với error**.

### Cấu hình composite index

```json
{
  "compositeIndexes": [
    [
      { "path": "/relevanceScore", "order": "descending" },
      { "path": "/uploadDate", "order": "descending" }
    ]
  ]
}
```

**Quy tắc về direction:**
- Composite index `(A DESC, B DESC)` cũng support `ORDER BY A ASC, B ASC` (cùng relative order)
- **KHÔNG** support mixed: `ORDER BY A DESC, B ASC` → cần composite index riêng

---

## Tối ưu Filter + Sort: Include filter property trong ORDER BY

```sql
-- Query gốc
SELECT * FROM c WHERE c.documentType = 'pdf' ORDER BY c.uploadDate DESC

-- Rewrite để dùng composite index
SELECT * FROM c
WHERE c.documentType = 'pdf'
ORDER BY c.documentType, c.uploadDate DESC
```

Rewrite cho phép dùng composite index `(documentType ASC, uploadDate DESC)` → giảm RU đáng kể.

```json
{
  "compositeIndexes": [
    [
      { "path": "/documentType", "order": "ascending" },
      { "path": "/uploadDate", "order": "descending" }
    ],
    [
      { "path": "/category", "order": "ascending" },
      { "path": "/relevanceScore", "order": "descending" }
    ]
  ]
}
```

---

## Composite Indexes cho Multi-property Filters

**Quy tắc design:**
1. Equality filters đặt **trước** trong composite index
2. Mỗi composite index support **tối đa một range filter** (đặt cuối)

```sql
SELECT * FROM c
WHERE c.category = 'reports'
  AND c.department = 'finance'
  AND c.createdDate > '2024-06-01'
```

```json
[
  { "path": "/category", "order": "ascending" },    // equality — đặt trước
  { "path": "/department", "order": "ascending" },  // equality — đặt trước
  { "path": "/createdDate", "order": "ascending" }  // range — đặt cuối
]
```

**Nếu có 2 range filters:** Cần 2 composite indexes riêng biệt, Cosmos DB dùng cả hai.

---

## Tuple Indexes — Array Element Filtering

Khi query filter theo **nhiều properties trong cùng một array element**:

```json
// Document
{
  "chunks": [
    { "position": 0, "text": "...", "tokens": 150 },
    { "position": 1, "text": "...", "tokens": 200 }
  ]
}
```

```sql
-- Filter theo cả position VÀ tokens trong cùng element
SELECT c.id, chunk.text
FROM c JOIN chunk IN c.chunks
WHERE chunk.position >= 0 AND chunk.tokens > 100
```

```json
// Tuple index
{
  "includedPaths": [
    { "path": "/chunks/[]/{position, tokens}/?" }
  ]
}
```

Dùng cho AI app lưu chunked documents — query theo chunk position, size, metadata.

---

## Index Transformation Behavior

Khi thay đổi indexing policy → Cosmos DB transform index **bất đồng bộ**:

| Action | Behavior |
|---|---|
| **Thêm index** | Query chưa dùng index mới cho đến khi transform xong |
| **Xoá index** | Query **ngay lập tức** ngừng dùng index đó → fall back to scan |
| **Thay index** | Thêm index mới trước → chờ transform xong → xóa index cũ |

**Best practice khi replace:** Add new → wait → remove old — đảm bảo queries luôn có index support.

---

## Balance Read vs Write Performance

Mỗi index tăng write latency và RU. Guidelines:

- **Include only queried properties** — nếu property chỉ được read (display), không index
- **Analyze query frequency** — index cho frequent queries, bỏ qua rare queries
- **Monitor write RU** — so sánh trước và sau khi thêm index
- **Use selective indexing** — `excludedPaths: /*` rồi include specific paths

---

## Bản chất bài này là gì?

**Một câu:** Composite index là bắt buộc cho `ORDER BY` nhiều property — thiếu thì query fail (không chỉ slow), và design rule là equality filter trước, range filter sau, mixed direction không support.

### So sánh với SQL composite index

| | SQL | Cosmos DB Composite |
|---|---|---|
| `ORDER BY a, b` | Cần composite index | Bắt buộc composite index (query fail nếu không có) |
| Mixed direction `ORDER BY a ASC, b DESC` | ✅ Flexible | ❌ Cần index riêng cho mixed direction |
| Equality trước range | Best practice | **Bắt buộc** theo thiết kế |
| Multiple range filters | Hạn chế | Tối đa 1 range filter per composite index |
| Transform khi thêm/xóa | Blocking (table lock) hoặc async | **Async** — add trước, xóa cũ sau khi xong |

**Transform behavior quan trọng cho zero-downtime:**
- Thêm index → transform async → query chưa dùng index mới ngay
- **Xóa index → query NGAY LẬP TỨC fall back to scan** (không chờ)
- Quy tắc: add new → wait transform complete → remove old. Không làm ngược lại.

**Tuple index cho AI chunking:** Khi lưu document chunks (`chunks: [{position, text, tokens}]`), cần filter theo position VÀ tokens trong cùng 1 chunk element → Tuple index. Đây là use case AI-specific.

---

## Checklist ghi nhớ cho AI-200

- [ ] Range index: `=`, `>`, `<`, `ORDER BY` single property, string functions
- [ ] Composite index bắt buộc cho **ORDER BY 2+ properties** — không có → query fail
- [ ] Composite index `(A DESC, B DESC)` cũng support `(A ASC, B ASC)` nhưng **KHÔNG** support mixed direction
- [ ] Design composite: **equality filters trước, range filter cuối**
- [ ] Mỗi composite index support **tối đa 1 range filter**
- [ ] Rewrite query: include filter property vào ORDER BY để dùng composite index
- [ ] Tuple index: `/arrayName/[]/{prop1, prop2}/?` — filter nhiều properties trong cùng array element
- [ ] Transform: thêm index mới → chờ transform → xóa index cũ (không ngược lại)
- [ ] Xóa index = query ngay lập tức fall back to scan

---

[← Bài 1](./cosmosdb-m3-bai1-indexes.md) · [🏠 Mục lục](../README.md) · [Bài 3 →](./cosmosdb-m3-bai3-vector-indexes-consistency.md)
