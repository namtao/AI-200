# Module Assessment — Build and Query with Azure Database for PostgreSQL

> Khoá: AI-200 · PostgreSQL — Build and query with Azure Database for PostgreSQL

---

## Câu 1

**Your AI application needs to store conversation metadata with varying structures. Some conversations include custom fields like model configuration, while others have user preferences. Which PostgreSQL data type best supports this requirement?**

- TEXT
- JSONB ✅
- VARCHAR(MAX)

**Đáp án: JSONB**

`JSONB` là binary JSON format hỗ trợ:
- Varying structure — mỗi document có fields khác nhau
- Nested objects và arrays
- Indexable (GIN index) → query JSONB content efficient
- Operators cho filtering: `@>`, `?`, `->`, `->>`

```sql
metadata JSONB DEFAULT '{}'::jsonb

-- Flexible: conversation A có model config, conversation B không có
-- {"model": "gpt-4", "temperature": 0.7}
-- {"user_pref": "dark_mode", "language": "en"}
```

Tại sao các đáp án còn lại sai:
- **TEXT:** Lưu raw text không có structure. Không thể query theo key/value, không index từng field trong JSON, phải parse toàn bộ string trong application code — inefficient và không scalable.
- **VARCHAR(MAX):** Không tồn tại trong PostgreSQL — đây là SQL Server syntax. PostgreSQL dùng `TEXT` (unlimited) hoặc `VARCHAR(n)`. Ngay cả nếu dùng TEXT, cũng không giải quyết được vấn đề varying structure.

---

## Câu 2

**You insert a new conversation record and need the auto-generated ID immediately to insert related messages. Which PostgreSQL clause retrieves the generated value without a separate query?**

- RETURNING ✅
- OUTPUT
- SELECT LAST_INSERT_ID()

**Đáp án: RETURNING**

```sql
INSERT INTO conversations (user_id, session_id)
VALUES ('user123', 'sess_abc')
RETURNING id;   -- Trả về generated ID trong cùng một round trip
```

Python:
```python
cur.execute(
    "INSERT INTO conversations (user_id, session_id) VALUES (%s, %s) RETURNING id",
    ('user123', 'sess_abc')
)
conversation_id = cur.fetchone()[0]   # Get generated ID ngay
```

Một round trip, không cần query riêng → quan trọng cho high-volume AI app.

Tại sao các đáp án còn lại sai:
- **OUTPUT:** SQL Server syntax (dùng trong T-SQL). Không tồn tại trong PostgreSQL.
- **SELECT LAST_INSERT_ID():** MySQL syntax. Trong PostgreSQL context, equivalent là `currval()` hoặc `RETURNING` — nhưng `LAST_INSERT_ID()` không phải PostgreSQL function. `RETURNING` là idiomatic PostgreSQL và an toàn hơn trong concurrent scenarios.

---

## Câu 3

**You need to insert a user preference record. If the record already exists, you want to update the value without creating a duplicate. Which PostgreSQL clause handles this scenario?**

- `INSERT INTO table ON DUPLICATE KEY UPDATE`
- `INSERT INTO table ON CONFLICT DO UPDATE` ✅
- `INSERT INTO table IF NOT EXISTS`

**Đáp án: ON CONFLICT DO UPDATE**

```sql
INSERT INTO user_preferences (user_id, preference_key, preference_value)
VALUES ('user123', 'theme', 'dark')
ON CONFLICT (user_id, preference_key)        -- Conflict trên unique constraint
DO UPDATE SET
    preference_value = EXCLUDED.preference_value,   -- EXCLUDED = inserted values
    updated_at = CURRENT_TIMESTAMP;
```

`EXCLUDED` pseudo-table chứa values sẽ được insert nếu không có conflict — reference để update.

Tại sao các đáp án còn lại sai:
- **ON DUPLICATE KEY UPDATE:** MySQL syntax. PostgreSQL không support — sẽ báo syntax error.
- **IF NOT EXISTS:** Không phải PostgreSQL DML syntax. `CREATE TABLE IF NOT EXISTS` tồn tại nhưng `INSERT IF NOT EXISTS` thì không. Nếu muốn ignore conflict: `ON CONFLICT DO NOTHING`.

---

## Câu 4

**Your Python application creates many short-lived database connections to store individual messages during AI inference. What approach should you improve performance?**

- Open a new connection for each message and close it immediately after
- Use a ConnectionPool to maintain reusable connections that the application can borrow and return ✅
- Keep a single global connection open for the entire application lifetime

**Đáp án: ConnectionPool**

```python
from psycopg_pool import ConnectionPool

pool = ConnectionPool(connection_string, min_size=2, max_size=20)

# Per request: borrow, use, return automatically
with pool.connection() as conn:
    with conn.cursor() as cur:
        cur.execute("INSERT INTO messages ...")
        conn.commit()
```

Pool reuses connections — không có connection establishment overhead mỗi request.

Tại sao các đáp án còn lại sai:
- **New connection per message:** Mỗi connection yêu cầu network handshake, authentication, server-side resource allocation. Với "many short-lived connections" (nhiều messages mỗi giây), overhead này dominate performance. Có thể exhaust server's max connections limit.
- **Single global connection:** Không thread-safe — nhiều threads/requests cùng dùng một connection gây race conditions. Nếu connection bị drop (network issue, server restart), toàn bộ app mất database access. Không scalable cho concurrent workload.

---

## Câu 5

**You're designing a table for AI agent task checkpoints. Tasks have a status that should only be 'pending', 'in_progress', 'completed', or 'failed'. Which constraint enforces this rule at the database level?**

- `UNIQUE (status)`
- `CHECK (status IN ('pending', 'in_progress', 'completed', 'failed'))` ✅
- `NOT NULL DEFAULT 'pending'`

**Đáp án: CHECK constraint**

```sql
CREATE TABLE task_checkpoints (
    id BIGSERIAL PRIMARY KEY,
    task_id BIGINT NOT NULL,
    status VARCHAR(20) NOT NULL CHECK (status IN ('pending', 'in_progress', 'completed', 'failed')),
    ...
);
```

PostgreSQL validate CHECK constraint khi INSERT hoặc UPDATE — bất kỳ giá trị nào ngoài list sẽ bị reject với error.

Tại sao các đáp án còn lại sai:
- **UNIQUE (status):** Chỉ đảm bảo không có duplicate status values trong table — với 4 allowed values, chỉ 4 rows được phép trong toàn table. Không phải ý nghĩa cần thiết ở đây.
- **NOT NULL DEFAULT 'pending':** Đảm bảo status không null và default là 'pending' — nhưng không ngăn insert `status = 'invalid_value'`. Cần CHECK để validate allowed values.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Data type cho varying structure metadata | JSONB |
| 2 | Lấy generated ID sau INSERT | RETURNING clause |
| 3 | Insert hoặc update nếu record exists | ON CONFLICT DO UPDATE |
| 4 | Many short-lived connections | ConnectionPool |
| 5 | Enforce allowed status values | CHECK (status IN (...)) |

---

*Module hoàn thành. PostgreSQL Learning Path tiếp theo: Module 2*
