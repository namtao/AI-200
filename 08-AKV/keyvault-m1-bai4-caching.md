# Bài 4 — Implement Caching Strategies to Reduce Key Vault Calls

> Khoá: AI-200 · Key Vault — Manage application secrets with Azure Key Vault

---

## Tại sao cần Caching?

Key Vault throttling limit: **4,000 GET transactions / vault / 10 giây** (per region). Write: 300 / 10s.

AI service 500 req/s không cache → 500 × 10 = 5,000 GET / 10s → exceed limit → **HTTP 429 Too Many Requests**.

Cache transforms pattern: thousands of vault calls → vài periodic refreshes.

---

## Pattern 1: Time-based In-memory Cache

```python
import time
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

class SecretCache:
    def __init__(self, vault_url, cache_ttl_seconds=900):    # 15 phút default
        credential = DefaultAzureCredential()
        self._client = SecretClient(vault_url=vault_url, credential=credential)
        self._cache = {}
        self._cache_ttl = cache_ttl_seconds

    def get_secret(self, secret_name):
        cached = self._cache.get(secret_name)
        now = time.monotonic()    # Không bị ảnh hưởng bởi system time changes

        if cached and (now - cached["timestamp"]) < self._cache_ttl:
            return cached["value"]    # Cache hit

        secret = self._client.get_secret(secret_name)
        self._cache[secret_name] = {"value": secret.value, "timestamp": now}
        return secret.value
```

TTL 900s → mỗi secret fetch tối đa 4 lần/giờ/instance. 10 replicas = 40 vault calls/giờ/secret.

---

## Pattern 2: Event Grid Invalidation

```
New secret version stored in KV
    → Event Grid: SecretNewVersionCreated
    → webhook / Azure Function / Service Bus
    → App invalidates local cache
    → Next access: fetch fresh value
```

Near-real-time freshness. **Defense-in-depth:** Dùng cả Event Grid + time-based TTL như fallback.

---

## Pattern 3: Startup Preloading

```python
class AppConfig:
    def __init__(self, vault_url, secret_names):
        credential = DefaultAzureCredential()
        client = SecretClient(vault_url=vault_url, credential=credential)

        self._secrets = {}
        for name in secret_names:
            self._secrets[name] = client.get_secret(name).value

    def get(self, secret_name):
        return self._secrets.get(secret_name)

# Startup: fetch once
config = AppConfig(
    vault_url="https://kv-ragpipeline-prod.vault.azure.net/",
    secret_names=["openai-api-key", "cosmosdb-connection-string", "blob-storage-key"]
)

# Request handling: no vault calls
api_key = config.get("openai-api-key")
```

Phù hợp cho: containerized services với rolling updates, secrets ít thay đổi.

---

## Cache Scope

| Scope | Description | Trade-off |
|---|---|---|
| **Per-process** | Each instance has own cache | Simple, scales với instances |
| **Shared (Redis)** | All instances share 1 cache | 1 vault call per refresh, more complex |
| **Startup preload** | Load all at start, lifetime of process | Simplest, needs periodic refresh |

**Starting point: per-process cache** cho hầu hết applications.

---

## TTL Guidelines

| Rotation frequency | Cache TTL |
|---|---|
| Daily/weekly | 5-15 phút + Event Grid |
| **Monthly/quarterly** | **30-60 phút** |
| Annual/static | Startup preloading + periodic refresh mỗi vài giờ |

API key rotate 90 days → TTL 1 giờ là reasonable (staleness 1h << 90-day rotation cycle).

---

## Handle Throttling (HTTP 429)

```python
def get_secret_with_monitoring(cache, secret_name):
    try:
        return cache.get_secret(secret_name)
    except HttpResponseError as e:
        if e.status_code == 429:
            logger.warning("Key Vault throttled. Consider increasing cache TTL.")
        raise
```

Exponential backoff: 1s → 2s → 4s → 8s → 16s + random jitter.

SDK has built-in retry policy. Log 429 events để tune cache TTL.

---

## Bản chất bài này là gì?

**Một câu:** Key Vault throttle limit là 4,000 GET/vault/10s — bất kỳ high-throughput service nào cũng phải cache secrets, với time-based TTL là starting point và Event Grid invalidation cho near-real-time freshness.

### So sánh 3 caching patterns

| | No Cache | Time-based TTL | Event Grid Invalidation | Startup Preload |
|---|---|---|---|---|
| **Vault calls** | 1 per request | ≤1 per TTL window per secret | 1 per rotation event | N lần khi start |
| **500 req/s** | 5,000/10s → 429 ngay | Vài lần/giờ | Vài lần/rotation | 0 khi running |
| **Freshness sau rotation** | Ngay lập tức | Tối đa 1 TTL delay | Near-real-time | Chỉ khi restart/refresh |
| **Complexity** | Thấp nhất | Thấp | Cao (Function + Event Grid) | Thấp |
| **Khi dùng** | Không nên | Starting point (hầu hết) | Critical secrets, fast rotation | Secrets ít đổi, rolling deploy |
| **Defense-in-depth** | N/A | Standalone | Kết hợp với TTL làm fallback | TTL cho periodic refresh |

**`time.monotonic()` thay vì `time.time()` cho cache TTL:** `time.time()` bị ảnh hưởng bởi system clock changes (NTP sync, DST). `time.monotonic()` tăng đều không bao giờ backward — đúng cho elapsed time measurement.

**HTTP 429 = tăng TTL, không phải retry ngay:** Khi thấy 429, giải pháp là tăng cache TTL. Retry ngay sẽ tiếp tục bị throttle. Exponential backoff chỉ là last resort — root fix là cache properly.

---

## Checklist ghi nhớ cho AI-200

- [ ] GET limit: **4,000 transactions / vault / 10 giây**
- [ ] HTTP **429** = throttling → implement caching
- [ ] Time-based cache: `time.monotonic()` (clock-drift safe), TTL 900s default
- [ ] Event Grid: `SecretNewVersionCreated` = near-real-time invalidation
- [ ] Event Grid + time-based TTL = defense-in-depth
- [ ] Startup preloading: load all secrets at start, no vault calls during requests
- [ ] Per-process cache = recommended starting point
- [ ] TTL = balance rotation frequency vs staleness acceptability

---

*Module Assessment tiếp theo*
