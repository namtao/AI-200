# Module Assessment — Implement Data Operations in Azure Managed Redis

> Khoá: AI-200 · Redis — Implement data operations in Azure Managed Redis

---

## Câu 1

**What is the default TLS connection port for Azure Managed Redis?**

- 10000 ✅
- 6379
- 6380

**Đáp án: 10000**

Azure Managed Redis và Enterprise caches dùng port **10000**. Đây là điểm khác biệt quan trọng so với Azure Cache for Redis (port 6380) và community Redis (port 6379).

```python
r = redis.Redis(
    host='your-redis-instance',
    port=10000,    # Azure Managed Redis
    ssl=True,
    decode_responses=True,
    password='your-access-key'
)
```

Tại sao các đáp án còn lại sai:
- **6379:** Port mặc định của community Redis (không có TLS). Azure yêu cầu TLS, không dùng 6379.
- **6380:** Port của **Azure Cache for Redis** (service khác). Azure Managed Redis là service riêng biệt với port khác (10000).

---

## Câu 2

**Which Redis command should you avoid using in production environments for iterating over keys?**

- KEYS ✅
- SCAN
- EXISTS

**Đáp án: KEYS**

`KEYS pattern` **block toàn bộ Redis server** trong khi thực thi — scan tất cả keys trong database. Với production database có hàng triệu keys, lệnh này có thể block server hàng giây, làm ứng dụng không thể phục vụ requests.

```python
# KHÔNG DÙNG trong production
r.keys('user:*')    # Blocks server

# DÙNG SCAN thay thế — non-blocking, iterate theo batches
cursor = 0
while True:
    cursor, keys = r.scan(cursor, match='user:*', count=100)
    if keys:
        r.delete(*keys)
    if cursor == 0:
        break
```

Tại sao các đáp án còn lại sai:
- **SCAN:** Đây chính xác là replacement cho KEYS. SCAN iterate qua keys theo cursor-based pagination — non-blocking, an toàn cho production.
- **EXISTS:** Kiểm tra sự tồn tại của key cụ thể (không scan patterns), không block server, hoàn toàn OK trong production.

---

## Câu 3

**Which redis-py method sets a key with an expiration time in a single atomic operation?**

- `setex()` ✅
- `expire()`
- `set()`

**Đáp án: `setex()`**

`setex(key, seconds, value)` thực hiện SET và EXPIRE trong **một atomic operation** — key được tạo và expiration được set đồng thời, không có khoảng thời gian nào mà key tồn tại mà không có TTL.

```python
r.setex('session:abc123', 60, 'user_data')    # Set + expire in 1 atomic op
```

Tại sao các đáp án còn lại sai:
- **`expire()`:** Add expiration cho key **đã tồn tại**. Cần 2 operations: `r.set(key, value)` rồi `r.expire(key, seconds)`. Giữa 2 operations, key tồn tại không có expiration — có race condition tiềm năng.
- **`set()`:** Basic set operation không set expiration. Có thể pass `ex=seconds` như argument (vd. `r.set(key, value, ex=60)`) nhưng đây không phải atomic theo cùng nghĩa — question hỏi về method **chuyên dùng** cho set + expire.

---

## Câu 4

**What does a TTL value of -1 indicate when checking key expiration in Redis?**

- Key exists but has no expiration set ✅
- Key doesn't exist
- Key expired 1 second ago

**Đáp án: Key exists but has no expiration set**

```python
ttl = r.ttl('user:1002:preferences')
```

Ba giá trị TTL đặc biệt:
- **-1:** Key **tồn tại** nhưng **không có expiration** (persistent key)
- **-2:** Key **không tồn tại**
- **N (dương):** Key tồn tại, còn **N giây** trước khi expire

```python
ttl = r.ttl('user:1002:preferences')
if ttl == -1:
    print("Key exists, no expiration")
elif ttl == -2:
    print("Key does not exist")
else:
    print(f"Expires in {ttl} seconds")
```

Tại sao các đáp án còn lại sai:
- **Key doesn't exist:** Đó là **-2**, không phải -1. Quan trọng phân biệt: -1 = exists without TTL, -2 = doesn't exist.
- **Key expired 1 second ago:** Redis không có giá trị TTL âm nào biểu thị "đã expire bao lâu" — key đã expire bị remove khỏi Redis, TTL query trả về -2 (không tồn tại).

---

## Tổng kết — Kết quả 4/4

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Default TLS port | 10000 |
| 2 | Command không dùng trong production | KEYS (dùng SCAN thay thế) |
| 3 | Set key + expiration atomically | `setex()` |
| 4 | TTL = -1 nghĩa là gì | Key exists, no expiration set |

---

*Module hoàn thành.*

---

[← Bài 3](./redis-m1-bai3-data-operations.md) · [🏠 Mục lục](../README.md) · [Bài 1 →](./redis-m2-bai1-pubsub.md)
