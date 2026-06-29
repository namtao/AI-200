# Bài 3 — Reference Key Vault Secrets from App Configuration

> Khoá: AI-200 · App Configuration — Manage application settings with Azure App Configuration

---

## Key Vault References là gì?

Special key-value pair trong App Configuration nơi value chứa **URI trỏ đến secret trong Key Vault**. Content type: `application/vnd.microsoft.appconfig.keyvaultref+json;charset=utf-8`.

- App Configuration lưu **pointer** (URI), không lưu actual secret value
- Provider resolve tự động khi load
- App code access bình thường qua dictionary syntax — không distinguish KV vs regular settings

---

## Tạo Key Vault Reference (CLI)

```bash
az appconfig kv set-keyvault \
    --name myAppConfigStore \
    --key "OpenAI:ApiKey" \
    --secret-identifier "https://my-keyvault.vault.azure.net/secrets/openai-api-key" \
    --label "Production"
```

- **Versionless URI** (recommended) → auto pick up latest version khi secret rotate
- Versioned URI → pinned đến specific version

Labels apply bình thường cho KV references → different vaults per environment.

---

## Provider Configuration với Key Vault Resolution

```python
from azure.appconfiguration.provider import load
from azure.identity import DefaultAzureCredential
import os

endpoint = os.environ.get("AZURE_APPCONFIG_ENDPOINT")
credential = DefaultAzureCredential()

config = load(
    endpoint=endpoint,
    credential=credential,
    keyvault_credential=credential    # Same credential cho cả App Config + Key Vault
)

# Unified access — app không biết source
model_endpoint = config["OpenAI:Endpoint"]    # Regular value từ App Config
api_key = config["OpenAI:ApiKey"]             # Resolved từ Key Vault
batch_size = config["Pipeline:BatchSize"]     # Regular value từ App Config
```

---

## Required Role Assignments

| Service | Role | Assign to |
|---|---|---|
| App Configuration | **`App Configuration Data Reader`** | App managed identity |
| Key Vault | **`Key Vault Secrets User`** | App managed identity |

Thiếu Key Vault role → provider throw error khi resolve reference (error message chỉ rõ vault/secret nào bị missing).

Multiple Key Vaults → identity cần `Key Vault Secrets User` trên **mỗi vault**. Hoặc dùng `keyvault_client_configs` để map vault URI → specific credentials.

---

## Secret Refresh Interval

```python
config = load(
    endpoint=endpoint,
    credential=credential,
    keyvault_credential=credential,
    refresh_on=[WatchKey("Sentinel")],
    refresh_interval=60,              # Check config changes mỗi 60s
    secret_refresh_interval=7200      # Re-resolve Key Vault secrets mỗi 2 giờ
)
```

`secret_refresh_interval` (seconds) = **độc lập** từ `refresh_interval`:
- Config settings check: mỗi 60s
- Key Vault secret re-resolve: mỗi 7200s (2h)

Alignment với rotation schedule. Mỗi KV reference = 1 request đến Key Vault → consider throttling limits.

---

## Workflow khi Secret Rotate

1. KV: old secret → new version (latest)
2. App Config: KV reference URI không đổi (versionless → auto latest)
3. `secret_refresh_interval` elapse → app call `config.refresh()`
4. Provider re-resolve reference → get latest version → updated secret available

---

## Bản chất bài này là gì?

**Một câu:** KV Reference là pointer pattern — App Configuration chỉ lưu URI trỏ đến secret trong Key Vault, provider resolve tự động khi `load()`, app code dùng dictionary syntax thống nhất mà không cần biết value đến từ đâu.

### So sánh với các cách khác để inject secrets vào app

| | Hard-code secret | Env var / K8s Secret | App Config direct | App Config + KV Reference |
|---|---|---|---|---|
| **Secret location** | Source code | Container env / Secret object | App Config store | Key Vault (pointer trong App Config) |
| **Audit per access** | Không | Không | Không | Có — Key Vault log từng lần read |
| **Rotation** | Redeploy | Restart pod | Manual update | Versionless URI → auto latest |
| **HSM encryption** | Không | Không | Không | Có (Premium tier) |
| **App code** | Hard-coded | `os.environ["KEY"]` | `config["Key"]` | `config["Key"]` — same interface |
| **Who manages secrets** | Developer | DevOps (K8s Secret) | DevOps (App Config) | Security team (KV) + DevOps (pointer) |

**`keyvault_credential` trong `load()` là bắt buộc:** Nếu thiếu, provider throw error khi gặp KV reference content type. Trong hầu hết cases, pass cùng `DefaultAzureCredential` instance cho cả `credential` và `keyvault_credential`.

**`secret_refresh_interval` và `refresh_interval` hoàn toàn độc lập:** Config settings check theo `refresh_interval` (60s), KV secret re-resolve theo `secret_refresh_interval` (7200s). Exam thường hỏi về sự khác biệt này.

---

## Checklist ghi nhớ cho AI-200

- [ ] KV reference: App Config stores **URI pointer**, không stores secret value
- [ ] Content type: `application/vnd.microsoft.appconfig.keyvaultref+json`
- [ ] `keyvault_credential=credential` trong `load()` để enable KV resolution
- [ ] Same `DefaultAzureCredential` cho cả App Config + Key Vault
- [ ] Required roles: **`App Configuration Data Reader`** + **`Key Vault Secrets User`**
- [ ] Versionless URI = recommended (auto pick up latest rotated secret)
- [ ] `secret_refresh_interval` = độc lập từ `refresh_interval`
- [ ] Labels apply to KV references → different vaults per env
- [ ] Single `load()` call = unified dictionary cho cả regular + KV secrets

---

*Bài tiếp theo: Bài 4 — Decide what to store in App Configuration vs Key Vault*
