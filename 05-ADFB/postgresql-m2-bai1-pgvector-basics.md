# Bài 1 — Store and Query Embeddings with pgvector

> Khoá: AI-200 · PostgreSQL — Implement vector search with Azure Database for PostgreSQL

---

## Enable pgvector

```sql
CREATE EXTENSION IF NOT EXISTS vector;

-- Verify
SELECT * FROM pg_extension WHERE extname = 'vector';
```

pgvector thêm: `vector` data type, distance operators, index types cho similarity search.

---

## Schema Design

```sql
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    content TEXT,
    category TEXT,
    practice_area TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    embedding vector(1536)    -- Dimension phải match embedding model
);
```

**Dimension phải match output của embedding model:**

| Model | Dimensions |
|---|---|
| Sentence transformers | 384 |
| `text-embedding-ada-002` | **1,536** |
| `text-embedding-3-large` | Đến 3,072 |

Sai dimension = insertion error.

### Multiple embeddings per document

```sql
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    content TEXT,
    title_embedding vector(384),      -- Title embedding
    content_embedding vector(1536)    -- Content embedding
);
```

---

## Insert và Update Embeddings

```sql
-- Insert với vector literal
INSERT INTO documents (title, content, category, embedding)
VALUES (
    'Corporate Merger Agreement Template',
    'This agreement outlines...',
    'contracts',
    '[0.0123, -0.0456, 0.0789, ...]'::vector
);

-- Batch insert
INSERT INTO documents (title, content, category, embedding)
VALUES
    ('Doc 1', 'Content...', 'legal', '[...]'::vector),
    ('Doc 2', 'Content...', 'legal', '[...]'::vector);

-- Update khi content thay đổi
UPDATE documents
SET content = 'Updated content...',
    embedding = '[0.0234, -0.0567, ...]'::vector,
    updated_at = NOW()
WHERE id = 42;
```

---

## Distance Operators

| Operator | Metric | Score range | Dùng khi |
|---|---|---|---|
| `<->` | L2 (Euclidean) | 0 = identical, larger = different | Vector magnitude quan trọng |
| `<=>` | Cosine distance | 0 = identical, 2 = opposite | **Text embeddings — most common** |
| `<#>` | Negative inner product | Smaller = more similar | Max inner product search |

**Cosine distance** (`<=>`) phổ biến nhất cho text embeddings vì đo góc (direction of meaning), không phụ thuộc magnitude.

```sql
-- Cosine similarity search
SELECT id, title, embedding <=> '[0.0123, -0.0456, ...]'::vector AS distance
FROM documents
ORDER BY distance
LIMIT 10;
```

---

## Vector Data Types

| Type | Precision | Storage (1536-dim) | Dùng khi |
|---|---|---|---|
| `vector` | 32-bit float | ~6 KB/row | Default, most use cases |
| `halfvec` | 16-bit float | ~3 KB/row | Storage concern, sau khi test quality |
| `sparsevec` | Sparse (non-zeros only) | Varies | Sparse embeddings |

```sql
-- halfvec cho compact storage
embedding halfvec(1536)

-- sparsevec
embedding sparsevec(10000)
```

> **HNSW index trên `sparsevec`** support tối đa **1,000 non-zero elements**. Vượt quá → cần dimensionality reduction.

---

## Checklist ghi nhớ cho AI-200

- [ ] Enable: `CREATE EXTENSION IF NOT EXISTS vector;`
- [ ] `vector(n)` — n phải match embedding model dimensions
- [ ] `text-embedding-ada-002` = **1,536 dimensions**
- [ ] Distance operators: `<=>` cosine · `<->` L2 · `<#>` inner product
- [ ] **`<=>` cosine distance** — recommended cho text embeddings
- [ ] Cosine: 0 = identical, 2 = opposite directions
- [ ] Vector literal: `'[0.1, -0.2, ...]'::vector`
- [ ] Data types: `vector` (32-bit) · `halfvec` (16-bit, 50% storage) · `sparsevec`
- [ ] Match operator class với distance operator khi tạo index

---

*Bài tiếp theo: Bài 2 — Perform fast vector similarity search*
