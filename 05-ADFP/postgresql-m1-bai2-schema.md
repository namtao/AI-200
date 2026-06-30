# Bài 2 — Create and Manage Schemas

> Khoá: AI-200 · PostgreSQL — Build and query with Azure Database for PostgreSQL

---

## Database và Schema Hierarchy

```
Server
└── Database (isolated, foreign keys không cross-database)
    └── Schema (namespace: default = public)
        └── Tables, functions, objects
```

**Dùng separate database khi:** Complete isolation cần thiết, different apps trên cùng server không nên thấy nhau.

**Dùng separate schema khi:** Related objects cần foreign key, cần JOIN across namespaces, logical separation.

**Hầu hết AI application:** Single database + default `public` schema là đủ.

---

## CREATE TABLE

```sql
CREATE TABLE conversations (
    id BIGSERIAL PRIMARY KEY,                        -- Auto-increment 64-bit
    session_id UUID NOT NULL,                        -- Required, unique per session
    user_id VARCHAR(255) NOT NULL,                   -- Required
    started_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    ended_at TIMESTAMP WITH TIME ZONE,               -- Nullable = conversation active
    metadata JSONB DEFAULT '{}'::jsonb               -- Flexible metadata
);
```

### Primary Key Types

| Type | Mô tả | Dùng khi |
|---|---|---|
| `SERIAL` | Auto-increment 32-bit (đến ~2 tỷ) | Small tables |
| `BIGSERIAL` | Auto-increment 64-bit | AI app với high insert volume |
| `UUID DEFAULT gen_random_uuid()` | Universally unique | Generate ID ngoài DB, merge from multiple sources |

---

## Data Types cho AI Application

| Type | Mô tả | Dùng khi |
|---|---|---|
| **JSONB** | Binary JSON, indexable, queryable | Flexible structure, varying fields, nested objects |
| `TEXT` | Unlimited string | Conversation content |
| `VARCHAR(n)` | String với max length | Enforce max length (vd. role VARCHAR(50)) |
| **TIMESTAMP WITH TIME ZONE** | Timestamp + timezone | Luôn dùng TIMESTAMPTZ cho temporal data |
| `BYTEA` | Binary data | Small binary objects |
| `BIGSERIAL` | Auto-increment | High-volume inserts |

> **TIMESTAMP WITH TIME ZONE (TIMESTAMPTZ)** — luôn dùng cho AI app có thể cross timezone. PostgreSQL lưu UTC, convert dựa trên session timezone.

---

## Constraints

```sql
-- NOT NULL: required field
user_id VARCHAR(255) NOT NULL

-- DEFAULT: giá trị mặc định
started_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP

-- CHECK: validate value
role VARCHAR(50) CHECK (role IN ('user', 'assistant', 'system'))
status VARCHAR(20) CHECK (status IN ('pending', 'in_progress', 'completed', 'failed'))

-- UNIQUE: no duplicates
email VARCHAR(255) UNIQUE

-- PRIMARY KEY = NOT NULL + UNIQUE
```

PostgreSQL check constraints khi INSERT hoặc UPDATE — catch error ở database level.

---

## Foreign Keys và Relationships

```sql
-- One-to-many: messages thuộc về conversation
CREATE TABLE messages (
    id BIGSERIAL PRIMARY KEY,
    conversation_id BIGINT NOT NULL REFERENCES conversations(id),
    role VARCHAR(50) NOT NULL CHECK (role IN ('user', 'assistant', 'system')),
    content TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

### Referential Actions (ON DELETE)

| Action | Behavior |
|---|---|
| `RESTRICT` (default) | Ngăn delete nếu có row referencing |
| `CASCADE` | Tự động delete tất cả rows referencing |
| `SET NULL` | Set FK về NULL |
| `SET DEFAULT` | Set FK về default value |

> `ON DELETE CASCADE` cẩn thận — xóa conversation xóa luôn tất cả messages.

### Many-to-many — Junction table

```sql
CREATE TABLE conversation_tags (
    conversation_id BIGINT REFERENCES conversations(id) ON DELETE CASCADE,
    tag_id BIGINT REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (conversation_id, tag_id)   -- Composite PK = no duplicates
);
```

---

## Indexes

```sql
-- B-tree index (default) — equality, range, ORDER BY
CREATE INDEX idx_messages_conversation_id ON messages(conversation_id);
CREATE INDEX idx_messages_created_at ON messages(created_at);

-- Composite index — queries filter on multiple columns
CREATE INDEX idx_messages_conversation_created ON messages(conversation_id, created_at);
```

**Composite index column order matters:** Index on `(conversation_id, created_at)` → satisfy queries filtering on `conversation_id` alone OR `conversation_id + created_at`. Không satisfy queries filtering chỉ trên `created_at`.

**Trade-off:** Index tốt cho reads, nhưng tăng overhead cho inserts/updates/deletes + tốn disk. AI app write nhiều (mỗi message) → chọn lọc index.

---

## Schema Management

```sql
-- Thêm column (efficient — PostgreSQL lưu default trong catalog, không update từng row)
ALTER TABLE conversations ADD COLUMN category VARCHAR(100);

-- Drop column
ALTER TABLE conversations DROP COLUMN old_field;

-- DROP TABLE (irreversible, bị block nếu có FK references)
DROP TABLE conversations CASCADE;  -- cascade drops FK constraints

-- Transaction cho schema changes
BEGIN;
ALTER TABLE conversations ADD COLUMN category VARCHAR(100);
CREATE INDEX idx_conversations_category ON conversations(category);
COMMIT;
-- ROLLBACK nếu có lỗi
```

Hầu hết DDL statements trong PostgreSQL là **transactional** — có thể ROLLBACK nếu cần.

---

## Bản chất bài này là gì?

**Một câu:** PostgreSQL schema design cho AI app là chọn đúng data types (BIGSERIAL, JSONB, TIMESTAMPTZ), đặt constraints đúng nơi (database level, không phải application level), và hiểu DDL transactional — khác với MySQL và SQL Server ở những điểm không hiển nhiên.

### So sánh với MySQL vs SQL Server vs MongoDB (document DB)

| | PostgreSQL | MySQL | SQL Server | MongoDB |
|---|---|---|---|---|
| DDL transactional | ✅ `BEGIN/COMMIT` DDL | ❌ Auto-commit DDL | ❌ Một số DDL auto-commit | N/A |
| Native JSON type | **JSONB** (binary, indexable) | JSON (text, limited index) | JSON (text functions) | Native document |
| Auto-increment | `BIGSERIAL` / `GENERATED` | `AUTO_INCREMENT` | `IDENTITY` | ObjectId |
| UUID gen | `gen_random_uuid()` built-in | UUID() | `NEWID()` | Built-in ObjectId |
| `ON DELETE CASCADE` | ✅ | ✅ | ✅ | N/A (denormalized) |
| CHECK constraints | ✅ Enforced | ✅ (từ 8.0.16) | ✅ | Validation rules |
| Composite index selectivity | Leftmost prefix rule | Leftmost prefix rule | Leftmost prefix rule | Compound index |

**PostgreSQL DDL là transactional — đây là điểm khác biệt lớn nhất:** Với MySQL và SQL Server, `ALTER TABLE` không thể ROLLBACK. PostgreSQL cho phép wrap toàn bộ schema migration trong `BEGIN/COMMIT` — nếu index creation fails, column addition cũng rollback. Critical cho zero-downtime migration.

**JSONB vs JSON (text) là khác biệt thực sự:** MongoDB lưu BSON natively, nhưng PostgreSQL JSONB cho phép dùng GIN index trên operators `@>`, `?` — không thể làm với TEXT hay VARCHAR. MySQL JSON type không có `@>` containment operator — phải dùng `JSON_CONTAINS()` function, không index efficient như JSONB với GIN.

---

## Checklist ghi nhớ cho AI-200

- [ ] `BIGSERIAL` cho high-volume AI app insert
- [ ] `UUID DEFAULT gen_random_uuid()` khi generate ID ngoài DB
- [ ] **JSONB** cho flexible metadata, varying structure, nested objects
- [ ] **TIMESTAMPTZ** (TIMESTAMP WITH TIME ZONE) cho temporal data
- [ ] `CHECK (status IN (...))` enforce allowed values tại database level
- [ ] Foreign key `REFERENCES table(column)` — RESTRICT, CASCADE, SET NULL, SET DEFAULT
- [ ] `ON DELETE CASCADE` cẩn thận — auto-delete cascading
- [ ] Composite index: column thứ nhất phải match filter để dùng được
- [ ] PostgreSQL DDL là transactional — wrap schema changes trong `BEGIN/COMMIT`

---

[← Bài 1](./postgresql-m1-bai1-explore-connect.md) · [🏠 Mục lục](../README.md) · [Bài 3 →](./postgresql-m1-bai3-query.md)
