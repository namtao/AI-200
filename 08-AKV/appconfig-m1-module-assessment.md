# Module Assessment — Manage Application Settings with Azure App Configuration

> Khoá: AI-200 · App Configuration — Manage application settings with Azure App Configuration

---

## Câu 1

**A developer loads App Configuration settings with two `SettingSelector` entries: one for the null label and one for the `Production` label. The key `Pipeline:BatchSize` exists with both labels. Which value does the application use?**

- The value from the null label because null labels always take priority as defaults.
- The value from the `Production` label because it's loaded second and overrides the null-labeled value for the same key. ✅
- The provider raises an error because the same key can't exist with multiple labels in a single load operation.

**Đáp án: Production label value (loaded second, overrides)**

```python
selects = [
    SettingSelector(key_filter="*", label_filter="\0"),          # Null label loaded FIRST
    SettingSelector(key_filter="*", label_filter="Production")   # Production loaded SECOND → wins
]
```

**Last selector wins** cho same key. Đây là composition pattern: defaults (null) load trước, env-specific overrides load sau.

Tại sao các đáp án còn lại sai:
- **Null label always takes priority:** Ngược lại. Null label là defaults vì không có env-specific override. Khi có Production label, Production value win vì loaded second.
- **Provider raises error:** Hoàn toàn sai. Đây là documented, intentional behavior — multiple SettingSelector cho cùng key là cách implement env differentiation trong App Configuration.

---

## Câu 2

**What does Azure App Configuration store when you create a Key Vault reference for a secret?**

- An encrypted copy of the secret value from Key Vault.
- A shared access signature (SAS) token that grants temporary access to the secret in Key Vault.
- A URI that points to the secret in Azure Key Vault, along with reference metadata and a specific content type. ✅

**Đáp án: URI pointer + reference metadata + content type**

App Configuration lưu **chỉ pointer** đến Key Vault, không phải actual secret value:
```json
{
    "uri": "https://my-keyvault.vault.azure.net/secrets/openai-api-key",
    "content_type": "application/vnd.microsoft.appconfig.keyvaultref+json;charset=utf-8"
}
```

Provider nhận content type → recognize as KV reference → resolve từ KV tự động. App Config **never** stores actual secret.

Tại sao các đáp án còn lại sai:
- **Encrypted copy:** Nếu App Config store copy, đây là two sources of truth → drift risk. Security model phụ thuộc vào App Config encryption chứ không phải Key Vault's HSM. Không đúng với actual implementation.
- **SAS token:** SAS tokens là temporary, expire. App Config references cần work indefinitely và pick up rotated secrets automatically. SAS không support this pattern.

---

## Câu 3

**A developer building an AI document processing pipeline needs to store the Azure OpenAI model deployment name (`gpt-4o`). Where should this setting be stored?**

- Azure Key Vault because all Azure OpenAI-related settings should be stored together with the API key for organizational consistency.
- Azure App Configuration as a regular key-value pair because the deployment name is nonsensitive and doesn't grant access to any resource. ✅
- Azure App Configuration as a Key Vault reference because the setting is related to a service that also has secrets.

**Đáp án: App Configuration (regular key-value)**

`gpt-4o` là model deployment name — **không grant access** to any resource. Biết tên deployment không cho phép gọi Azure OpenAI API (vẫn cần API key). Đây là non-sensitive operational setting phù hợp với App Configuration.

Benefits: có thể dùng labels để switch model per environment, feature flags để test new model versions, dynamic refresh để update deployment name mà không redeploy.

Tại sao các đáp án còn lại sai:
- **Key Vault for organizational consistency:** Placement decision dựa trên sensitivity, không phải organizational proximity. Key Vault cho non-sensitive settings = wasted security overhead, no labels, no feature flags, lower throttle limits.
- **App Config as KV reference:** KV references chỉ dành cho values stored IN Key Vault. Deployment name không phải secret và không cần Key Vault's security features. Creating a KV reference cho non-sensitive value là unnecessary complexity.

---

## Câu 4

**Which two Azure RBAC role assignments does an application's managed identity need to resolve Key Vault references from App Configuration?**

- App Configuration Data Reader on the App Configuration store and Key Vault Secrets User on the Key Vault. ✅
- App Configuration Data Owner on the App Configuration store and Key Vault Administrator on the Key Vault.
- Key Vault Secrets User on both the App Configuration store and the Key Vault.

**Đáp án: App Config Data Reader + Key Vault Secrets User**

2 services = 2 role assignments:
- **`App Configuration Data Reader`** trên App Config store → read settings và KV references
- **`Key Vault Secrets User`** trên Key Vault → read secret values that references point to

```python
config = load(
    endpoint=endpoint,
    credential=credential,          # App Config Data Reader role
    keyvault_credential=credential  # Key Vault Secrets User role
)
```

Tại sao các đáp án còn lại sai:
- **Data Owner + Key Vault Administrator:** Quá broad. Data Owner cho phép write settings — app runtime chỉ cần read. Key Vault Administrator cũng quá broad (tất cả data plane ops). Least privilege principle bị vi phạm.
- **Key Vault Secrets User on both:** App Configuration không dùng Key Vault RBAC roles. App Config là separate service với own RBAC (`App Configuration Data Reader`/`Data Owner`). Applying KV role đến App Config resource không có effect.

---

## Câu 5

**A developer enables dynamic configuration refresh with a sentinel key and a 60-second refresh interval. What must happen for the application to pick up configuration changes from the store?**

- The provider automatically pushes updated values to the application whenever any key changes in the store.
- The application must restart to load the latest configuration because the provider caches all values at the initial `load()` call.
- The application must call the `refresh()` method on the configuration object, and the sentinel key must have changed since the last refresh check. ✅

**Đáp án: Call `refresh()` + sentinel key must have changed**

```python
config = load(
    endpoint=endpoint,
    credential=DefaultAzureCredential(),
    refresh_on=[WatchKey("Sentinel")],
    refresh_interval=60
)

# Trong request handler hoặc app loop
config.refresh()    # Phải gọi explicitly
```

2 điều kiện cần thiết:
1. `config.refresh()` được gọi explicitly (provider không push changes tự động)
2. `Sentinel` key đã thay đổi kể từ lần check cuối (nếu không thay đổi → refresh không xảy ra)
3. `refresh_interval` (60s) phải đã elapsed kể từ lần check cuối

Pattern: Operator update settings → update `Sentinel` key → next `refresh()` call → reload tất cả.

Tại sao các đáp án còn lại sai:
- **Provider automatically pushes:** App Configuration là pull model, không push. Provider không maintain persistent connection để receive push notifications. Phải explicitly poll via `refresh()`.
- **Must restart to load:** Đây chính xác là điều dynamic refresh giải quyết. Dynamic refresh cho phép update config mà không restart. Nếu cần restart, tại sao gọi là "dynamic refresh"?

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | SettingSelector stacking | Production label (last selector wins) |
| 2 | KV reference stores what | URI pointer + metadata + content type |
| 3 | Where to store deployment name | App Config (non-sensitive, no access grant) |
| 4 | Required roles for KV reference resolution | App Config Data Reader + KV Secrets User |
| 5 | Dynamic refresh requirements | Call `refresh()` + sentinel key changed |

---

*Module hoàn thành. Learning Path kết thúc.*
