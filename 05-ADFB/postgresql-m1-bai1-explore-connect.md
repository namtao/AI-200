# Bài 1 — Explore Azure Database for PostgreSQL + Connect

> Khoá: AI-200 · PostgreSQL — Build and query with Azure Database for PostgreSQL

---

## Azure Database for PostgreSQL là gì?

Fully managed relational database service chạy community edition PostgreSQL. Microsoft quản lý: hardware provisioning, software patching, backup, high availability.

**Lợi ích với AI application:** Kết hợp transactional reliability, flexible data modeling (JSONB), và extension ecosystem bao gồm **pgvector** cho vector similarity search.

---

## Architecture

Compute và storage **tách biệt** — independent scaling, built-in data durability qua locally redundant storage.

### 3 Compute Tiers

| Tier | VM series | Phù hợp |
|---|---|---|
| **Burstable** | B-series | Dev, test, workload không cần full CPU liên tục, intermittent |
| **General Purpose** | D-series | Production, web app, backend service, predictable performance |
| **Memory Optimized** | E-series | Large in-memory dataset, complex analytical queries, AI in-memory computation |

Có thể thay đổi tier sau deployment (restart ngắn).

---

## Managed Service Capabilities

**Backup:**
- Tự động create và lưu trên zone-redundant storage (hoặc locally redundant nếu region không support AZ)
- Default retention: **7 ngày**, extend đến **35 ngày**
- AES 256-bit encryption
- Point-in-time restore đến bất kỳ giây nào trong retention period → tạo server instance mới

**PgBouncer — Built-in connection pooler:**
- Maintain pool of reusable connections → giảm overhead tạo connection mới
- Connect trên port **6432** (thay vì 5432)
- Enable qua server configuration

> **Quan trọng:** PgBouncer chỉ có trên **General Purpose** và **Memory Optimized**. Burstable tier **không hỗ trợ**.

---

## Extensions cho AI

| Extension | Chức năng |
|---|---|
| **pgvector** | Vector data types + similarity search — RAG, semantic search |
| **pg_trgm** | Trigram text similarity — fuzzy matching, autocomplete |
| **hstore** | Key-value pairs trong PostgreSQL value |

Enable bằng: `CREATE EXTENSION pgvector;`

---

## Connection Fundamentals

Server endpoint: `<server-name>.postgres.database.azure.com`

Core parameters:
- **host** — server FQDN
- **port** — 5432 (direct) hoặc 6432 (PgBouncer)
- **database** — tên database
- **username** — `username@servername` với Entra auth
- **password** — static password hoặc Entra token
- **sslmode** — SSL configuration

---

## Authentication Methods

### Microsoft Entra Authentication (Recommended)

OAuth 2.0 tokens thay vì password. Lợi ích:
- Centralized identity management
- Token short-lived, không cần lưu password
- Hỗ trợ managed identities
- Audit trails trong Entra sign-in logs

```python
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
token = credential.get_token("https://ossrdbms-aad.database.windows.net/.default")
# Dùng token.token làm password trong connection string
```

`DefaultAzureCredential` tự động chọn credential:
- Azure: Managed Identity
- Local dev: Azure CLI credentials

### PostgreSQL Native Authentication

Username/password truyền thống. Dùng khi: legacy app không support Entra, non-Azure environment.

Best practices: lưu password trong **Azure Key Vault**, rotate định kỳ, strong random passwords.

---

## TLS/SSL Configuration

Azure yêu cầu TLS mặc định (TLS 1.2 và 1.3). Các `sslmode`:

| Mode | Encryption | Certificate validation | Production? |
|---|---|---|---|
| `disable` | ❌ | ❌ | Azure reject |
| `allow` | Nếu server yêu cầu | ❌ | |
| `prefer` | Nếu server support | ❌ | |
| `require` | ✅ | ❌ | |
| `verify-ca` | ✅ | ✅ CA | |
| **`verify-full`** | ✅ | ✅ CA + hostname | **Production** |

**`verify-full` cho production** — xác nhận certificate signed bởi trusted CA VÀ hostname match.

---

## Connection String Examples

```
# Basic
postgresql://user:password@server.postgres.database.azure.com/db?sslmode=require

# Full certificate verification
postgresql://user:password@server.postgres.database.azure.com/db?sslmode=verify-full&sslrootcert=/etc/ssl/certs/ca-certificates.crt

# PgBouncer (port 6432)
postgresql://user:password@server.postgres.database.azure.com:6432/db?sslmode=require
```

---

## Checklist ghi nhớ cho AI-200

- [ ] 3 tiers: **Burstable** (dev) · **General Purpose** (production) · **Memory Optimized** (AI, analytics)
- [ ] PgBouncer chỉ có **General Purpose** và **Memory Optimized**, port **6432**
- [ ] Backup default retention: **7 ngày**, max **35 ngày**, AES-256
- [ ] Point-in-time restore tạo **server mới**, không restore in-place
- [ ] Extensions: `pgvector` (vector search) · `pg_trgm` (fuzzy text) · `hstore` (key-value)
- [ ] Entra auth: token từ `https://ossrdbms-aad.database.windows.net/.default`
- [ ] `DefaultAzureCredential`: Managed Identity (Azure) hoặc Azure CLI (local)
- [ ] `sslmode=verify-full` cho production
- [ ] Direct: port **5432** · PgBouncer: port **6432**

---

*Bài tiếp theo: Bài 2 — Create and manage schemas*
