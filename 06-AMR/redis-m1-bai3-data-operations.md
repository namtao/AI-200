# Bài 3 — Implement Data Operations

> Khoá: AI-200 · Redis — Implement data operations in Azure Managed Redis

---

## Redis Data Types

| Type | Commands | Dùng khi |
|---|---|---|
| **String** | SET, GET, MSET, MGET | Text, serialized data |
| **Hash** | HSET, HGET, HGETALL | Structured objects (user profile, product) |
| **List** | LPUSH, RPUSH, LPOP, RPOP, LRANGE | Queue, activity feed, recent items |
| **Numeric** | INCR, DECR, INCRBY, DECRBY | Counters, rate limiting (atomic, no race condition) |

---

## Basic Operations

### String (SET/GET)

```python
r.set('user:1001:name', 'Alice Smith')      # Returns True
name = r.get('user:1001:name')              # Returns 'Alice Smith'
```

### Multi-key String (MSET/MGET)

```python
r.mset({
    'user:1001:name': 'Alice Smith',
    'user:1001:email': 'alice@example.com',
    'user:1001:age': '28'
})

values = r.mget('user:1001:name', 'user:1001:email', 'user:1001:age')
# Returns list: ['Alice Smith', 'alice@example.com', '28']
```

### Hash (HSET/HGET/HGETALL)

```python
r.hset('user:1001', mapping={
    'name': 'Alice Smith',
    'email': 'alice@example.com',
    'age': '28'
})

name = r.hget('user:1001', 'name')          # Single field
user_data = r.hgetall('user:1001')          # All fields → dict
```

Hash hiệu quả hơn nhiều keys riêng lẻ cho structured objects.

### Pipeline cho batch operations

```python
pipe = r.pipeline()
pipe.hgetall('user:1001')
pipe.hgetall('user:1002')
results = pipe.execute()    # Một round trip, 2 results
```

### Check existence và Delete

```python
# EXISTS — tất cả data types
if r.exists('user:1001:name'):
    print("Key exists")

count = r.exists('user:1001:name', 'user:1001:email', 'user:9999:name')
# Returns 2 (số keys tồn tại)

# DELETE
result = r.delete('user:1001:name')          # Returns số keys đã xoá
r.delete('user:1001:email', 'user:1001:age')
```

---

## Expiration (TTL)

### SETEX — Set value + expiration atomically

```python
r.setex('session:abc123', 60, 'user_data')     # Expire sau 60 giây
r.psetex('temp:data', 5000, 'temp')            # Expire sau 5000 milliseconds
```

### EXPIRE — Add expiration cho existing key

```python
r.set('user:1002:preferences', 'dark_mode')   # Không có expiration
r.expire('user:1002:preferences', 3600)        # Expire sau 1 giờ
r.pexpire('user:1002:preferences', 3600000)    # Expire sau 3600000ms
r.expireat('user:1002:preferences', unix_timestamp)  # Expire tại timestamp cụ thể
```

### TTL — Check expiration

```python
ttl = r.ttl('user:1002:preferences')
# -1 = key exists, NO expiration set
# -2 = key does NOT exist
# N  = expires in N seconds
```

```python
r.persist('user:1002:preferences')    # Remove expiration (make permanent)
```

---

## Cache Invalidation Patterns

### 1. Time-based (TTL tự động)

```python
def cache_query(query_key, query_function, ttl=300):
    cached = r.get(query_key)
    if cached:
        return cached                          # Cache HIT

    result = query_function()                  # Cache MISS
    r.setex(query_key, ttl, result)
    return result
```

### 2. Manual invalidation khi data update

```python
def update_user_profile(user_id, new_data):
    save_to_database(user_id, new_data)

    cache_keys = [
        f'user:{user_id}:profile',
        f'user:{user_id}:summary',
        f'dashboard:user:{user_id}'
    ]
    deleted = r.delete(*cache_keys)    # Delete tất cả related keys
```

### 3. Pattern-based (SCAN — không dùng KEYS)

```python
def invalidate_user_cache(user_id):
    pattern = f'user:{user_id}:*'
    cursor = 0

    while True:
        cursor, keys = r.scan(cursor, match=pattern, count=100)
        if keys:
            r.delete(*keys)
        if cursor == 0:
            break    # Scan hoàn tất
```

---

## TTL Guidelines

| Data type | TTL |
|---|---|
| Frequently changing | 1-5 phút |
| Moderate change rate | 15-60 phút |
| Relatively stable | 1-24 giờ |
| Static reference data | 24+ giờ |

---

## Bản chất bài này là gì?

**Một câu:** Redis cung cấp 4 data types chính với TTL/expiration management để implement cache-aside pattern — key insight là biết khi nào dùng Hash thay nhiều String keys.

### So sánh String vs Hash cho structured data

| | Nhiều String keys | Hash |
|---|---|---|
| Lưu user profile | `user:1:name`, `user:1:email`, `user:1:age` | `user:1` với mapping |
| Memory | Cao (overhead mỗi key) | Thấp hơn |
| Atomic get all fields | Cần MGET | HGETALL |
| Partial update | Overwrite key | HSET field cụ thể |
| Phù hợp | Scalar values | Structured objects |

**EXISTS trả về số lượng, không phải True/False:** `r.exists('k1', 'k2', 'k3')` trả về 0, 1, 2, hoặc 3 — không phải boolean. Đây là exam trap thường gặp.

**TTL -1 vs -2:** `-1` = key tồn tại, không có expiration; `-2` = key không tồn tại. Hai giá trị âm này dễ bị nhầm lẫn.

---

## Checklist ghi nhớ cho AI-200

- [ ] **String** SET/GET · **Hash** HSET/HGET/HGETALL · **List** LPUSH/RPUSH/LPOP · **Numeric** INCR/DECR
- [ ] Hash hiệu quả hơn nhiều keys riêng lẻ cho structured data
- [ ] Pipeline = batch commands trong một round trip
- [ ] `EXISTS` trả về **số lượng** keys tồn tại (không phải True/False)
- [ ] `r.setex(key, seconds, value)` = set + expire atomically
- [ ] `r.expire(key, seconds)` = add expiration cho existing key
- [ ] TTL values: **-1** = no expiration · **-2** = key không tồn tại · **N** = còn N giây
- [ ] `r.persist(key)` = remove expiration
- [ ] Cache invalidation: TTL auto · Manual delete · Pattern SCAN
- [ ] **SCAN** thay vì KEYS cho pattern matching trong production

---

*Module Assessment tiếp theo*

---

[← Bài 2](./redis-m1-bai2-client-libraries.md) · [🏠 Mục lục](../README.md) · [Assessment →](./redis-m1-module-assessment.md)
