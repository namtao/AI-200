# Module Assessment — Manage Application Secrets with Azure Key Vault

> Khoá: AI-200 · Key Vault — Manage application secrets with Azure Key Vault

---

## Câu 1

**Your AI application needs to read secrets from Azure Key Vault at runtime but shouldn't be able to create or delete secrets. Which built-in RBAC role should you assign to the application's managed identity?**

- Key Vault Secrets User ✅
- Key Vault Secrets Officer
- Key Vault Contributor

**Đáp án: Key Vault Secrets User**

`Key Vault Secrets User` = **read-only** access to secret values. App managed identity có thể `get_secret()` tại runtime nhưng không thể `set_secret()`, `delete_secret()`, hay list/manage secrets.

Tại sao các đáp án còn lại sai:
- **Key Vault Secrets Officer:** Full CRUD on secrets — create, update, delete, list. Quá broad cho app runtime access. Assign cho operators và CI/CD pipelines quản lý secret lifecycle, không phải application code.
- **Key Vault Contributor:** Đây là **control plane** role — quản lý vault resource (tạo, xóa vault, update vault properties). **KHÔNG** grant data plane access (đọc secrets, keys, certificates). Đây là common misconception — tên "Contributor" có vẻ powerful nhưng với Key Vault nó không giúp đọc secrets.

---

## Câu 2

**You want your application to authenticate with Key Vault using managed identity in production and Azure CLI credentials during local development, without changing code between environments. Which credential class should you use?**

- ManagedIdentityCredential
- EnvironmentCredential
- DefaultAzureCredential ✅

**Đáp án: DefaultAzureCredential**

`DefaultAzureCredential` chains multiple authentication methods theo thứ tự:
- Production: `ManagedIdentityCredential` — dùng managed identity của Azure compute resource
- Local dev: `AzureCliCredential` — dùng account từ `az login`

Same code, không cần modify giữa environments.

```python
credential = DefaultAzureCredential()    # Works everywhere
client = SecretClient(vault_url=vault_url, credential=credential)
```

Tại sao các đáp án còn lại sai:
- **ManagedIdentityCredential:** Chỉ works trong production (Azure compute với managed identity). Local dev không có managed identity → fail. Cần separate code paths cho local dev.
- **EnvironmentCredential:** Đọc `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_CLIENT_SECRET` env vars. Works cho service principal authentication nhưng không automatically switch sang Azure CLI local. Requires explicit env var setup cho cả hai environments.

---

## Câu 3

**You store three versions of a secret named 'cosmosdb-connection-string' in Key Vault. What does calling `get_secret('cosmosdb-connection-string')` without a version parameter return?**

- The first version that was created
- The latest enabled version of the secret ✅
- All three versions as a list of KeyVaultSecret objects

**Đáp án: The latest enabled version**

`get_secret()` không có version parameter → **latest enabled version** tự động. Điều này cho phép rotation đơn giản: tạo new version, app refetch → nhận new credential mà không cần biết version GUID.

```python
# Returns latest enabled version (version 3 nếu có 3 versions)
latest = client.get_secret("cosmosdb-connection-string")

# Retrieve specific version
specific = client.get_secret("cosmosdb-connection-string",
                              version="abc123def456...")
```

Tại sao các đáp án còn lại sai:
- **First version:** Sẽ là ngược lại với mong muốn. Nếu trả về first version, rotation sẽ không có tác dụng — app vẫn dùng old credential ngay cả khi new version đã tạo.
- **All three versions as a list:** `get_secret()` trả về `KeyVaultSecret` object, không phải list. Để list versions dùng `list_properties_of_secret_versions("cosmosdb-connection-string")` — trả về metadata của tất cả versions nhưng không trả về values.

---

## Câu 4

**Your AI application connects to a service that supports two active keys simultaneously. You need to rotate the credential without any application downtime. Which rotation strategy should you use?**

- Dual-credential rotation ✅
- Manual rotation with application restart
- Automated rotation with Event Grid only

**Đáp án: Dual-credential rotation**

Với services hỗ trợ 2 active keys (Azure Storage, Cosmos DB):

1. Generate new secondary key trên target service
2. Store new key as new secret version trong KV
3. Wait cho tất cả app instances pick up new version
4. Regenerate old primary key (invalidate old)

→ **Luôn có ít nhất 1 valid credential** trong rotation window → zero downtime.

Tại sao các đáp án còn lại sai:
- **Manual rotation with restart:** Application restart = downtime. "Without any downtime" requirement không được đáp ứng. Manual cũng có human error risk trong urgent rotation (4-hour requirement).
- **Automated rotation with Event Grid only:** Event Grid automation giải quyết "khi nào rotate" nhưng không giải quyết "làm sao không bị downtime." Nếu service chỉ support 1 active key, automated rotation vẫn có window khi old key invalid và new key chưa picked up.

---

## Câu 5

**Your AI inference service processes 500 requests per second and retrieves an API key from Key Vault for each request. The API key rotates every 90 days. What caching approach best balances performance with security?**

- No caching with direct Key Vault calls per request
- Time-based in-memory cache with a one-hour TTL ✅
- Startup preloading with no periodic refresh

**Đáp án: Time-based in-memory cache với 1-hour TTL**

500 req/s = 5,000 GET / 10s → **exceed KV limit của 4,000 GET / 10s** → immediate throttling nếu không cache.

1-hour TTL + 90-day rotation cycle:
- KV calls: tối đa 24 calls/day/instance thay vì 43.2 million/day
- Staleness window: tối đa 1 giờ (acceptable relative to 90 days)
- Rotation transparency: app tự refresh khi TTL expire

Tại sao các đáp án còn lại sai:
- **No caching:** 500 × 4,000 / 4,000 = instant throttle. 429 errors everywhere. Unacceptable latency (tens of ms per KV call × 500 req/s). Completely impractical.
- **Startup preloading với no refresh:** Works ban đầu nhưng key rotates every 90 days. Nếu app chạy >90 days mà không restart và không refresh → sử dụng expired/invalid credential sau rotation. Không có periodic refresh mechanism → stale credentials indefinitely.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | RBAC role cho app runtime access | Key Vault Secrets User |
| 2 | Auth: managed identity prod, CLI local | DefaultAzureCredential |
| 3 | `get_secret()` không version | Latest enabled version |
| 4 | Rotate với 2 active keys, zero downtime | Dual-credential rotation |
| 5 | 500 req/s, key rotates 90 days | Time-based cache TTL 1 giờ |

---

*Module hoàn thành.*
