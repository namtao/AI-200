# Bài 3 — Query Data

> Khoá: AI-200 · PostgreSQL — Build and query with Azure Database for PostgreSQL

---

## SQL Execution Order — Tại sao alias không dùng được trong WHERE?

SQL thực thi theo thứ tự logic khác với thứ tự viết:

| Thứ tự | Clause | Mục đích |
|---|---|---|
| 1 | FROM | Identify source tables |
| 2 | WHERE | Filter rows |
| 3 | GROUP BY | Group rows |
| 4 | HAVING | Filter groups |
| 5 | **SELECT** | Choose columns, compute expressions (alias defined here) |
| 6 | ORDER BY | Sort (có thể dùng alias) |
| 7 | LIMIT/OFFSET | Restrict count |

**Alias scope:** Column aliases trong SELECT chỉ visible cho clauses sau SELECT (ORDER BY, LIMIT). WHERE, GROUP BY, HAVING thực thi trước SELECT → **không thể dùng alias**.

```sql
-- FAIL: WHERE thực thi trước SELECT, msg_date chưa tồn tại
SELECT DATE(created_at) AS msg_date FROM messages
WHERE msg_date > '2024-01-01';    -- Error: column "msg_date" does not exist

-- ĐÚNG: lặp lại expression trong WHERE
SELECT DATE(created_at) AS msg_date FROM messages
WHERE DATE(created_at) > '2024-01-01'
ORDER BY msg_date;    -- OK: ORDER BY sau SELECT, dùng được alias
```

---

## PostgreSQL-Specific Filtering

```sql
-- ILIKE: case-insensitive LIKE (không cần LOWER())
SELECT * FROM messages WHERE content ILIKE '%error%';

-- NULLS LAST/FIRST trong ORDER BY
ORDER BY ended_at NULLS LAST        -- active conversations cuối
ORDER BY completed_at NULLS FIRST   -- unprocessed tasks đầu

-- COALESCE: first non-null value
SELECT COALESCE(title, 'Untitled') FROM conversations;
```

---

## JSONB Operators

```sql
-- -> : extract JSON element (trả về JSON)
-- ->> : extract as text (cho comparison, display)
metadata->>'status'             -- status field as text
metadata->'config'              -- config field as JSON

-- #> / #>> : nested path
checkpoint_data#>>'{results,0,score}'   -- navigate nested path as text

-- ? : key existence
WHERE metadata ? 'priority'

-- @> : containment (JSON document chứa pattern này không?)
WHERE checkpoint_data @> '{"status": "completed"}'

-- Array: jsonb_array_elements
SELECT DISTINCT c.*
FROM conversations c,
     jsonb_array_elements_text(c.metadata->'tags') AS tag
WHERE tag = 'support';
```

`@>` và `?` có thể dùng **GIN index** cho efficient filtering trên large tables.

---

## Keyset Pagination (Cursor-based)

OFFSET pagination chậm với large tables — phải scan và discard skipped rows.

**Keyset pagination** dùng WHERE clause để skip → O(1) per page regardless of depth:

```sql
-- Page 1: 20 messages mới nhất
SELECT id, conversation_id, content, created_at
FROM messages
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Page tiếp theo: filter từ last seen values
SELECT id, conversation_id, content, created_at
FROM messages
WHERE (created_at, id) < ('2024-06-15 10:30:00', 12345)  -- row tuple comparison
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Include `id` trong cả ORDER BY và WHERE để handle ties (nhiều rows cùng timestamp).

App lưu last row's sort values, pass vào query tiếp theo.

---

## Common Table Expressions (CTEs)

```sql
-- Non-recursive CTE: break complex query thành steps
WITH recent_conversations AS (
    SELECT id, user_id, started_at
    FROM conversations
    WHERE started_at > CURRENT_DATE - INTERVAL '7 days'
),
message_stats AS (
    SELECT conversation_id, COUNT(*) AS message_count, MAX(created_at) AS last_message_at
    FROM messages
    GROUP BY conversation_id
)
SELECT rc.user_id, rc.started_at, COALESCE(ms.message_count, 0) AS message_count
FROM recent_conversations rc
LEFT JOIN message_stats ms ON rc.id = ms.conversation_id;
```

**Recursive CTE** — query tree/hierarchical data:

```sql
WITH RECURSIVE task_tree AS (
    -- Base case (anchor)
    SELECT id, parent_id, title, 1 AS depth
    FROM tasks WHERE id = 1

    UNION ALL

    -- Recursive case
    SELECT t.id, t.parent_id, t.title, tt.depth + 1
    FROM tasks t
    INNER JOIN task_tree tt ON t.parent_id = tt.id
    WHERE tt.depth < 10    -- Depth limit để tránh infinite loop
)
SELECT * FROM task_tree ORDER BY depth, id;
```

Luôn có **depth limit** hoặc termination condition trong recursive CTE.

---

## INSERT với RETURNING

Lấy generated values sau INSERT/UPDATE/DELETE **trong một round trip**:

```sql
-- Get generated ID
INSERT INTO conversations (user_id, session_id)
VALUES ('user123', 'sess_abc')
RETURNING id;

-- Get multiple generated values
INSERT INTO messages (conversation_id, role, content)
VALUES (1, 'user', 'Hello')
RETURNING id, created_at;

-- Dùng với UPDATE
UPDATE tasks SET status = 'completed', completed_at = CURRENT_TIMESTAMP
WHERE id = 5
RETURNING id, status, completed_at;
```

Không cần query riêng để get ID — quan trọng cho high-volume AI app.

---

## Upserts với ON CONFLICT

```sql
-- Insert hoặc update nếu conflict
INSERT INTO user_preferences (user_id, preference_key, preference_value)
VALUES ('user123', 'theme', 'dark')
ON CONFLICT (user_id, preference_key)
DO UPDATE SET
    preference_value = EXCLUDED.preference_value,   -- EXCLUDED = values that would have been inserted
    updated_at = CURRENT_TIMESTAMP;

-- Ignore conflict (no-op)
INSERT INTO tags (name) VALUES ('important')
ON CONFLICT (name) DO NOTHING;

-- Biết row mới hay updated: (xmax = 0) = true khi inserted
INSERT INTO task_checkpoints (task_id, step_number, checkpoint_data)
VALUES (:task_id, :step_number, :data::jsonb)
ON CONFLICT (task_id, step_number)
DO UPDATE SET checkpoint_data = EXCLUDED.checkpoint_data, updated_at = CURRENT_TIMESTAMP
RETURNING id, (xmax = 0) AS is_new;
```

`EXCLUDED` pseudo-table tham chiếu đến values sẽ được insert nếu không có conflict.

---

## Checklist ghi nhớ cho AI-200

- [ ] SQL execution order: FROM → WHERE → GROUP BY → HAVING → **SELECT** → ORDER BY → LIMIT
- [ ] Column alias chỉ visible trong **ORDER BY** và sau (không dùng được trong WHERE)
- [ ] `ILIKE` = case-insensitive LIKE, không cần LOWER()
- [ ] `NULLS LAST` / `NULLS FIRST` trong ORDER BY
- [ ] JSONB operators: `->>` (text), `->` (JSON), `@>` (containment), `?` (key exists)
- [ ] `#>>'{nested,path}'` navigate JSONB nested path
- [ ] Keyset pagination: WHERE tuple `(col1, col2) < (last1, last2)` — không dùng OFFSET
- [ ] Recursive CTE: `WITH RECURSIVE` + UNION ALL + **depth limit**
- [ ] `RETURNING` clause — get generated values sau INSERT/UPDATE trong một round trip
- [ ] `ON CONFLICT DO UPDATE` = upsert · `ON CONFLICT DO NOTHING` = ignore
- [ ] `EXCLUDED` = values that would have been inserted
- [ ] `(xmax = 0) AS is_new` — biết row được insert hay updated

---

*Bài tiếp theo: Bài 4 — Integrate SDKs and applications*
