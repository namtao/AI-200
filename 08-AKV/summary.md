# Tổng hợp — Azure App Configuration & Key Vault

> Khoá: AI-200 · 08-AKV

---

## Bức tranh toàn cảnh: App Config vs Key Vault

```
Application
    ↓ load() một lần
    App Configuration
        ├── Regular settings: endpoint URL, deployment name, batch size, queue name
        ├── Feature flags: UseNewEmbeddingsModel, EnableNewPipeline
        └── KV References (URI pointer) ──→ Key Vault
                                                ├── openai-api-key
                                                ├── cosmosdb-connection-string
                                                └── blob-storage-key
    ↓
    Unified dictionary (app không phân biệt source)
```

**Quy tắc phân loại một câu:** "Value này có thể grant access đến resource không?"
- **Yes → Key Vault** (API key, connection string với credentials, storage key)
- **No → App Configuration** (endpoint URL, model name, batch size, timeout)

---

## Khi nào dùng cái nào — Decision Criteria

| Tình huống | Service | Lý do |
|---|---|---|
| `https://my-openai.openai.azure.com/` (endpoint) | App Config | Biết URL không grant access |
| `gpt-4o` (deployment name) | App Config | Không grant access |
| `sk-abc123` (API key) | Key Vault | Grant access đến OpenAI API |
| `AccountEndpoint=https://...;AccountKey=...` | Key Vault | Connection string chứa credentials |
| `Pipeline:BatchSize = 200` | App Config | Operational setting |
| Toggle processing path (feature flag) | App Config | Feature Management feature |
| TLS certificate + private key | Key Vault | Sensitive crypto material |
| Logging level, queue name | App Config | Non-sensitive operational |

---

## App Configuration — Nội dung cốt lõi

### Data Model

| Property | Details |
|---|---|
| **Key** | Case-sensitive, hierarchical với `:` hoặc `/` (vd. `OpenAI:Endpoint`) |
| **Value** | Unicode string ≤10 KB (combined key+value) |
| **Label** | Optional tag tạo variants của cùng key (`Development`, `Production`) |
| Null label | Keys không có label — `"\0"` trong provider API |

### Connect và Load

```python
from azure.appconfiguration.provider import load, SettingSelector
from azure.identity import DefaultAzureCredential

config = load(
    endpoint="https://<name>.azconfig.io",
    credential=DefaultAzureCredential(),
    selects=[
        SettingSelector(key_filter="*", label_filter="\0"),          # null label = defaults
        SettingSelector(key_filter="*", label_filter="Production")   # overrides (last wins)
    ],
    trim_prefixes=["DocPipeline:"],
    refresh_on=[WatchKey("Sentinel")],
    refresh_interval=60
)

config.refresh()    # Phải gọi explicitly trong request handler
value = config["OpenAI:Endpoint"]
```

### Feature Flags

```python
from featuremanagement import FeatureManager

config = load(
    endpoint=endpoint,
    credential=DefaultAzureCredential(),
    feature_flags_enabled=True,
    feature_flag_refresh_enabled=True,
    refresh_on=[WatchKey("Sentinel")],
    refresh_interval=30
)

feature_manager = FeatureManager(config)
if feature_manager.is_enabled("UseNewEmbeddingsModel"):
    process_with_new_model(document)
```

### KV References trong App Config

```python
config = load(
    endpoint=endpoint,
    credential=credential,
    keyvault_credential=credential,    # Bắt buộc cho KV resolution
    refresh_interval=60,
    secret_refresh_interval=7200       # Độc lập từ refresh_interval
)

api_key = config["OpenAI:ApiKey"]      # Transparent — app không biết đến từ KV
```

### RBAC — App Configuration

| Role | Permissions | Assign to |
|---|---|---|
| `App Configuration Data Reader` | Read-only | App managed identity |
| `App Configuration Data Owner` | Full CRUD | Operators, CI/CD |

---

## Key Vault — Nội dung cốt lõi

### 3 Object Types

| Type | Lưu gì | AI use case |
|---|---|---|
| **Secrets** | Arbitrary string ≤25 KB | API keys, connection strings, passwords |
| **Keys** | Cryptographic keys (server-side ops, never leaves vault) | Encrypt data, sign artifacts |
| **Certificates** | X.509 certs + private keys | HTTPS, mutual TLS |

### Connect và Read

```python
from azure.keyvault.secrets import SecretClient
from azure.identity import DefaultAzureCredential

client = SecretClient(
    vault_url="https://<vault-name>.vault.azure.net/",
    credential=DefaultAzureCredential()
)

secret = client.get_secret("openai-api-key")
value = secret.value
version = secret.properties.version    # GUID

# List metadata only (không reveal values)
for prop in client.list_properties_of_secrets():
    print(prop.name, prop.enabled)
```

### Versioning

```python
# Mỗi set_secret() cùng tên = new GUID version
client.set_secret("cosmosdb-connection-string", "new-value")

# get_secret() không version = latest ENABLED version
latest = client.get_secret("cosmosdb-connection-string")

# Specific version
specific = client.get_secret("cosmosdb-connection-string", version="abc123...")
```

### Rotation Strategies

| Strategy | Downtime | Complexity | Yêu cầu |
|---|---|---|---|
| Manual | Có (restart) | Thấp | Bất kỳ service |
| Automated (Event Grid `SecretNearExpiry` → Function) | Có thể | Cao | Bất kỳ service |
| **Dual-credential** | **Không** | Trung bình | Service hỗ trợ 2 active keys (Storage, Cosmos DB) |

### Caching Patterns

```python
class SecretCache:
    def __init__(self, vault_url, cache_ttl_seconds=900):    # 15 phút
        self._client = SecretClient(vault_url=vault_url, credential=DefaultAzureCredential())
        self._cache = {}
        self._cache_ttl = cache_ttl_seconds

    def get_secret(self, name):
        cached = self._cache.get(name)
        now = time.monotonic()    # clock-drift safe
        if cached and (now - cached["timestamp"]) < self._cache_ttl:
            return cached["value"]
        secret = self._client.get_secret(name)
        self._cache[name] = {"value": secret.value, "timestamp": now}
        return secret.value
```

**Throttle limit:** 4,000 GET transactions / vault / 10 giây. 500 req/s không cache = 5,000/10s → HTTP 429 ngay lập tức.

### RBAC — Key Vault

| Role | Permissions | Assign to |
|---|---|---|
| **`Key Vault Secrets User`** | Read secret values (data plane) | App managed identity |
| Key Vault Secrets Officer | Full CRUD secrets | Operators, CI/CD |
| Key Vault Administrator | All data plane ops | Admin only |
| Key Vault Reader | Metadata only (names, no values) | Monitoring |
| **`Key Vault Contributor`** | **Control plane only — KHÔNG đọc secrets** | Resource management |

---

## CLI Quick Reference

```bash
# App Configuration
az appconfig create --name myStore --resource-group myRG --location eastus --sku Standard
az appconfig kv set --name myStore --key "Pipeline:BatchSize" --value "200" --label Production
az appconfig kv set-keyvault --name myStore --key "OpenAI:ApiKey" \
    --secret-identifier "https://my-kv.vault.azure.net/secrets/openai-key" --label Production
az appconfig feature set --name myStore --feature UseNewEmbeddingsModel --label Production

# Key Vault
az keyvault create --name myKV --resource-group myRG --location eastus
az keyvault secret set --vault-name myKV --name openai-api-key --value "sk-abc123"
az keyvault secret show --vault-name myKV --name openai-api-key

# RBAC Assignments
az role assignment create --role "App Configuration Data Reader" \
    --assignee <managed-identity-principal-id> --scope <appconfig-resource-id>
az role assignment create --role "Key Vault Secrets User" \
    --assignee <managed-identity-principal-id> --scope <keyvault-resource-id>
```

---

## Exam Traps (≥8)

### 1. `Key Vault Contributor` không đọc được secrets
Tên nghe có vẻ powerful nhưng đây là **control plane role** — chỉ manage vault resource (tạo/xóa vault). Để đọc secrets cần `Key Vault Secrets User` (data plane). Assign `Key Vault Contributor` cho app = app vẫn bị Forbidden khi `get_secret()`.

### 2. Last selector wins, không phải first
Khi load null label (defaults) trước rồi load Production label sau, **Production label wins** cho cùng key. Thứ tự SettingSelector list quyết định precedence — later = higher priority. Exam hay hỏi "which value does the app use?" với 2 selectors.

### 3. Null label trong code là `"\0"` không phải `None`
`label_filter="\0"` trong SettingSelector match keys không có label. `label_filter=None` không filter theo null label. Đây là gotcha khi implement composition pattern.

### 4. `config.refresh()` phải gọi explicitly — không tự động
App Configuration là **pull model**. Provider không maintain persistent connection hay receive push notifications. Dynamic refresh chỉ xảy ra khi code gọi `config.refresh()` explicitly. Nếu không gọi, config không bao giờ update dù sentinel key đã thay đổi.

### 5. `secret_refresh_interval` và `refresh_interval` hoàn toàn độc lập
`refresh_interval` = kiểm tra config settings. `secret_refresh_interval` = re-resolve KV secrets. Hai clock riêng biệt. Config settings có thể check mỗi 60s nhưng KV secrets chỉ re-fetch mỗi 7200s. Cả hai triggered bởi cùng `config.refresh()` nhưng chạy theo schedule riêng.

### 6. Feature flag refresh độc lập từ config refresh — nhưng cùng trigger
`feature_flag_refresh_enabled=True` phải set riêng — nếu không set, FF không refresh dù `config.refresh()` được gọi. Tuy nhiên, khi đã set, cả FF và config đều refresh khi `config.refresh()` được gọi — không có `config.refresh_feature_flags()` riêng.

### 7. `get_secret()` không version → latest ENABLED, không phải latest version
Nếu latest version bị disable (`enabled=False`), get_secret() trả về version enabled gần nhất trước đó. Đây là cơ chế rollback — disable bad version để app tự động fall back.

### 8. Expiration date của secret là informational — không hard-block access
Expired secret **vẫn accessible** qua `get_secret()`. Expiry chỉ là metadata để trigger automation (Event Grid `SecretExpired`). App vẫn đọc được expired secret nếu không có code kiểm tra `expires_on`.

### 9. App Config KV reference lưu URI pointer, không phải copy secret
App Configuration **không bao giờ** lưu actual secret value. Chỉ lưu URI + content type metadata. Provider resolve khi `load()`. Nếu exam hỏi "what does App Config store for a KV reference?" → URI + content type, không phải encrypted copy hay SAS token.

### 10. `Key Vault Contributor` vs `Key Vault Secrets User` cho managed identity
App managed identity cần `Key Vault Secrets User` để đọc secrets. `Key Vault Contributor` (resource management) không giúp ích. Exam thường present "Key Vault Contributor" như option vì tên nghe "đủ quyền" — đây là distractor.

---

## Assessment Summaries

### App Configuration Module Assessment (5/5)

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | 2 SettingSelector (null + Production) cho cùng key → dùng value nào? | Production label (last selector wins) |
| 2 | App Config lưu gì khi tạo KV reference? | URI pointer + content type metadata (không phải secret value) |
| 3 | `gpt-4o` deployment name → lưu ở đâu? | App Config regular key-value (non-sensitive, không grant access) |
| 4 | Managed identity cần roles gì để resolve KV references? | App Config Data Reader + Key Vault Secrets User |
| 5 | Dynamic refresh với sentinel key — điều kiện để pick up changes? | Call `refresh()` explicitly + sentinel key phải đã thay đổi |

**Key insight:** Câu 5 có 3 điều kiện cần đủ: (1) `config.refresh()` called explicitly, (2) sentinel key đã thay đổi, (3) `refresh_interval` đã elapsed.

### Key Vault Module Assessment (5/5)

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | App cần read secrets, không create/delete → role nào? | Key Vault Secrets User (read-only data plane) |
| 2 | Auth với managed identity prod + CLI local, không đổi code → dùng gì? | DefaultAzureCredential (tự chain) |
| 3 | 3 versions stored, gọi `get_secret()` không version → trả về gì? | Latest enabled version |
| 4 | Service support 2 active keys, cần zero downtime rotation → strategy nào? | Dual-credential rotation |
| 5 | 500 req/s, API key rotate 90 ngày → caching approach tốt nhất? | Time-based in-memory cache, TTL 1 giờ |

**Key insight:** Câu 5: No caching = 5,000 GET/10s > limit 4,000 → 429 ngay. Startup preload không có periodic refresh = stale sau 90 ngày nếu app không restart.

---

## Decision Matrix: .env vs K8s ConfigMap+Secret vs App Config vs Key Vault

| Criterion | .env file | K8s ConfigMap + Secret | App Configuration | Key Vault |
|---|---|---|---|---|
| **Non-sensitive config** | .env | ConfigMap | App Config | Không phù hợp |
| **Sensitive credentials** | .env (insecure) | K8s Secret (base64, not encrypted) | KV Reference | Key Vault |
| **Multi-environment** | Separate files | Separate namespaces | Labels trên cùng key | Separate vaults |
| **Dynamic update không restart** | Không | Không (cần restart) | Có (dynamic refresh) | Không built-in |
| **Feature flags** | env var thủ công | env var thủ công | Built-in FeatureManager | Không |
| **Audit per secret access** | Không | Không | Không | Có |
| **HSM encryption** | Không | Không | Không | Có (Premium) |
| **Secret rotation** | Thủ công | Thủ công | Qua KV reference | Versioning + Event Grid |
| **Throttle limit** | N/A | N/A | Cao | 4,000 GET/vault/10s |
| **Best for** | Local dev | Containerized apps K8s | Centralized non-sensitive + feature flags | All sensitive credentials |
| **Anti-pattern** | Commit to repo | Secret không encrypt etcd | Lưu secrets trực tiếp | Lưu non-sensitive settings |

### Khi nào chọn gì

**Chỉ dùng .env:** Local dev only, không commit lên git.

**K8s ConfigMap + Secret:** App deploy trên K8s, không cần dynamic refresh, team quen K8s RBAC. Lưu ý: K8s Secret là base64, không phải encrypted — enable etcd encryption hoặc kết hợp với Key Vault.

**App Configuration:** Production apps cần centralized config, multi-environment, feature flags, dynamic refresh mà không restart. Không lưu secrets trực tiếp.

**Key Vault:** Mọi sensitive credential trong production. Kết hợp với App Config via KV references cho unified access pattern.

**App Configuration + Key Vault (recommended):** Single `load()` trong code, App Config là control plane cho config surface, Key Vault là security plane cho secrets. Labels, feature flags áp dụng cho toàn bộ.

---

*File tổng hợp — AI-200 Module 08-AKV*

---

[← Assessment](./keyvault-m1-module-assessment.md) · [🏠 Mục lục](../README.md) · [Mục lục →](../README.md)
