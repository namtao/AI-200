# Bài 4 — Integrate SDKs and Applications

> Khoá: AI-200 · PostgreSQL — Build and query with Azure Database for PostgreSQL

---

## Python Integration với psycopg

**psycopg (version 3)** là recommended PostgreSQL adapter cho Python.

```bash
pip install "psycopg[binary]"   # Binary distribution, không cần cài libpq
```

### Tạo connection

```python
import psycopg

# Connection string
conn = psycopg.connect("postgresql://user:pass@server.postgres.database.azure.com/db?sslmode=require")

# Individual parameters (flexible, compute values programmatically)
conn = psycopg.connect(
    host="server.postgres.database.azure.com",
    dbname="mydb",
    user="myuser",
    password="mypassword",
    sslmode="require"
)
```

### Context manager — ensure proper cleanup

```python
with psycopg.connect(connection_string) as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT * FROM conversations WHERE id = %s", (conversation_id,))
        row = cur.fetchone()
```

Context manager đóng connection đúng cách kể cả khi exception xảy ra.

---

## Parameterized Queries — Bắt buộc

```python
# ĐÚNG — parameterized, prevent SQL injection
cur.execute(
    "SELECT * FROM conversations WHERE user_id = %s AND status = %s",
    (user_id, status)
)

# Named parameters
cur.execute(
    "SELECT * FROM messages WHERE conversation_id = %(conv_id)s",
    {"conv_id": conv_id}
)

# KHÔNG BAO GIỜ dùng string formatting/concatenation với user input
# NGUY HIỂM: f"SELECT * FROM conversations WHERE user_id = '{user_id}'"
```

`%s` cho positional, `%(name)s` cho named parameters.

---

## Fetch Results

```python
row = cur.fetchone()           # Một row, None nếu không có
rows = cur.fetchall()          # Tất cả rows (memory!)
rows = cur.fetchmany(100)      # Batch của N rows

# Iterate trực tiếp (memory efficient cho large result set)
for row in cur:
    process(row)
```

**Large result set:** iterate trực tiếp thay vì `fetchall()` để tránh load tất cả vào memory.

---

## Connection Management Best Practices

### Timeouts

```python
conn = psycopg.connect(
    connection_string,
    connect_timeout=10,               # Giây chờ establish connection
    options="-c statement_timeout=30000"  # Milliseconds cho query timeout
)
```

Web app: 5-30 giây. Batch job: có thể dài hơn.

### Retry với exponential backoff

```python
import time
from psycopg import OperationalError

def connect_with_retry(connection_string, max_retries=3):
    for attempt in range(max_retries):
        try:
            return psycopg.connect(connection_string)
        except OperationalError as e:
            if attempt == max_retries - 1:
                raise
            wait = 2 ** attempt  # Exponential backoff: 1s, 2s, 4s
            time.sleep(wait)
```

**Retry cho:** `OperationalError` (connection, timeout, transient failure).
**KHÔNG retry:** Constraint violations, syntax errors — cần fix code.

---

## Error Handling

```python
from psycopg.errors import (
    UniqueViolation,
    ForeignKeyViolation,
    CheckViolation,
    DeadlockDetected
)

try:
    conn.execute("INSERT INTO users (email) VALUES (%s)", (email,))
    conn.commit()

except UniqueViolation:
    conn.rollback()
    return "Email already exists"

except ForeignKeyViolation:
    conn.rollback()
    return "Referenced record not found"

except CheckViolation:
    conn.rollback()
    return "Invalid value"

except DeadlockDetected:
    conn.rollback()
    # Retry — PostgreSQL auto-detects deadlock, terminates one transaction
    return retry_operation()
```

**Deadlock:** PostgreSQL detect và terminate một transaction. App phải rollback và retry. Giảm deadlock: acquire locks theo consistent order.

---

## Performance — Batch Inserts

```python
# executemany: insert nhiều rows — giảm network round trips
cur.executemany(
    "INSERT INTO messages (conversation_id, role, content) VALUES (%s, %s, %s)",
    records  # List of tuples
)

# COPY: fastest cho large datasets (10K+ rows) — 2-10x faster than individual inserts
with cur.copy("COPY messages (conversation_id, role, content) FROM STDIN") as copy:
    for record in records:
        copy.write_row(record)
```

---

## Connection Pooling

Tạo connection mới là expensive (network handshake, authentication, server-side allocation). Reuse connections bằng pool:

```python
from psycopg_pool import ConnectionPool

# Tạo pool một lần lúc app startup
pool = ConnectionPool(connection_string, min_size=1, max_size=10)

# Borrow và return connection
with pool.connection() as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT * FROM messages WHERE conversation_id = %s", (id,))
```

**Không dùng:**
- New connection per request → expensive, có thể exhaust connections
- Single global connection → blocking, không thread-safe

**Dùng:** Connection pool → reusable connections, configurable size.

---

## Summary Pattern cho AI App

```python
import psycopg
from psycopg_pool import ConnectionPool
from azure.identity import DefaultAzureCredential

# Setup lúc app startup
credential = DefaultAzureCredential()

def get_token():
    token = credential.get_token("https://ossrdbms-aad.database.windows.net/.default")
    return token.token

# Connection string với Entra token (refresh token khi expire)
connection_string = f"postgresql://user@server.postgres.database.azure.com/db?sslmode=require&password={get_token()}"

pool = ConnectionPool(connection_string, min_size=2, max_size=20)

# Per request
def save_message(conversation_id: int, role: str, content: str) -> int:
    with pool.connection() as conn:
        with conn.cursor() as cur:
            cur.execute(
                "INSERT INTO messages (conversation_id, role, content) VALUES (%s, %s, %s) RETURNING id",
                (conversation_id, role, content)
            )
            row = cur.fetchone()
            conn.commit()
            return row[0]
```

---

## Checklist ghi nhớ cho AI-200

- [ ] `psycopg` v3 là recommended Python adapter, install `psycopg[binary]`
- [ ] **Luôn dùng parameterized queries** — `%s` positional, `%(name)s` named
- [ ] **Không dùng string formatting** với user input — SQL injection risk
- [ ] Context manager (`with`) đảm bảo connection cleanup kể cả khi exception
- [ ] `fetchone()` một row · `fetchall()` tất cả · iterate cursor cho large set
- [ ] `connect_timeout` (giây) + `statement_timeout` (milliseconds)
- [ ] Retry với exponential backoff cho `OperationalError`, không retry constraint/syntax error
- [ ] `DeadlockDetected`: rollback và retry
- [ ] `executemany()` cho vài trăm đến vài nghìn rows
- [ ] `COPY` command: nhanh nhất cho 10K+ rows (2-10x faster)
- [ ] **ConnectionPool** (`min_size`, `max_size`) — không tạo connection per request

---

*Module Assessment tiếp theo*
