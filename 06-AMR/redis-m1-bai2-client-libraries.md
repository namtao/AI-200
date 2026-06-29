# Bài 2 — Client Libraries and Development Best Practices

> Khoá: AI-200 · Redis — Implement data operations in Azure Managed Redis

---

## Client Libraries

| Library | Language |
|---|---|
| StackExchange.Redis | C#/.NET |
| Lettuce | Java |
| node_redis | Node.js |
| **redis-py** | **Python** |

Luôn dùng latest version, upgrade thường xuyên.

---

## Ports

| Service | Port |
|---|---|
| **Azure Managed Redis / Enterprise cache** | **10000** |
| Azure Cache for Redis | 6380 |

---

## Clustering Policy

**Enterprise clustering policy** → tất cả client libraries đều work.

**OSS clustering policy** → cần client library support clustered Redis (vd. `redis.cluster.RedisCluster` thay vì `redis.Redis`).

---

## Connect với redis-py

### Basic connection (password)

```python
import redis

r = redis.Redis(
    host='your-redis-instance',
    port=10000,
    ssl=True,
    decode_responses=True,    # Convert bytes → strings automatically
    password='your-access-key'
)
```

`decode_responses=True` → text data. `decode_responses=False` → binary data (images, pickled objects).

### Microsoft Entra ID (production)

```python
from azure.identity import DefaultAzureCredential
from redis_entraid.cred_provider import create_from_default_azure_credential

credential_provider = create_from_default_azure_credential(
    ("https://redis.azure.com/.default",),
)

r = redis.Redis(
    host='your-redis-instance',
    port=10000,
    ssl=True,
    decode_responses=True,
    credential_provider=credential_provider
)
```

---

## Blocked Commands

Azure quản lý Redis instance → một số commands bị blocked:

**Enterprise clustering policy blocked:**
- `CLUSTER INFO`, `CLUSTER HELP`, `CLUSTER KEYSLOT`, `CLUSTER NODES`, `CLUSTER SLOTS`

**Active geo-replication blocked:**
- `FLUSHALL`, `FLUSHDB`

---

## Multi-key Commands và Clustering

**OSS clustering:** Tất cả keys phải map đến cùng hash slot → CROSSSLOT exception nếu không.

**Enterprise clustering:** Cho phép `DEL`, `MSET`, `MGET`, `EXISTS`, `UNLINK`, `TOUCH` across slots.

**Active-Active database:** Chỉ `MGET`, `EXISTS`, `TOUCH` work across slots.

---

## Development Best Practices

### Data Design
- Dùng **smaller values** — chia data lớn thành nhiều keys nhỏ
- Dùng **pipelining** — batch multiple commands trong một round trip
- **SCAN** thay vì `KEYS` — KEYS block server, SCAN non-blocking

### Network
- **Co-locate** Redis và app trong cùng Azure region
- Dùng **hostname** thay vì IP (IP có thể thay đổi khi scale)
- **TLS required** (1.2 và 1.3 supported)

### Monitoring — Scale khi metrics vượt **75%**
- Used Memory Percentage
- CPU usage
- Connected Clients
- Network bandwidth

### Configuration
- **High availability mode** cho production (không disable trừ dev/test)
- Enable **data persistence** cho quick recovery

---

## Bản chất bài này là gì?

**Một câu:** Kết nối đúng vào Azure Managed Redis đòi hỏi port 10000, TLS bắt buộc, clustering policy quyết định class nào dùng, và production cần Entra ID thay password.

### So sánh Enterprise vs OSS Clustering Policy

| | Enterprise Clustering | OSS Clustering |
|---|---|---|
| Client class | `redis.Redis` | `redis.cluster.RedisCluster` |
| Multi-key ops | DEL, MSET, MGET, EXISTS cross slots | Chỉ MGET, EXISTS, TOUCH |
| Cross-slot | Cho phép | CROSSSLOT exception |
| Phù hợp | Hầu hết trường hợp | Khi cần Redis OSS compatibility |

**Port 10000 là exam trap:** Azure Managed Redis = 10000, Azure Cache for Redis = 6380, Community Redis = 6379. Ba service khác nhau, ba port khác nhau.

**Exam trap — SCAN vs KEYS:** `KEYS` block toàn bộ server trong production. Với hàng triệu keys, `KEYS *` có thể block Redis hàng giây. Luôn dùng `SCAN` với cursor-based pagination.

---

## Checklist ghi nhớ cho AI-200

- [ ] Python library: **redis-py**
- [ ] Azure Managed Redis port: **10000** (không phải 6380)
- [ ] `decode_responses=True` → text · `False` → binary
- [ ] Enterprise clustering → tất cả libraries work
- [ ] OSS clustering → cần RedisCluster class
- [ ] FLUSHALL/FLUSHDB blocked khi active geo-replication
- [ ] **SCAN** thay vì `KEYS` trong production (KEYS blocks server)
- [ ] Co-locate Redis và app trong cùng region
- [ ] Scale khi metrics **> 75%**
- [ ] TLS required (1.2 và 1.3)

---

*Bài tiếp theo: Bài 3 — Implement data operations*
