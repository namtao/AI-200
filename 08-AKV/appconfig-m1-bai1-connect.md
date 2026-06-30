# Bài 1 — Connect to App Configuration from Application Code

> Khoá: AI-200 · App Configuration — Manage application settings with Azure App Configuration

---

## Azure App Configuration là gì?

Fully managed service lưu application settings (non-sensitive) dưới dạng key-value pairs trong centralized cloud store. **Tách settings khỏi code** — update config không cần rebuild/redeploy.

---

## Data Model

| Property | Description |
|---|---|
| **Key** | Case-sensitive unicode string. Hierarchical với `:` hoặc `/` (vd. `OpenAI:Endpoint`) |
| **Value** | Unicode string ≤10 KB (combined key+value). Plain string, JSON, hoặc KV reference |
| **Label** | Optional tag tạo **variants của cùng key** (vd. `Development`, `Production`) |
| **Content type** | Cách interpret value (`application/json`, KV reference content type) |
| **Tags** | Optional metadata key-value pairs |

Key không có label → **null label** (default).

---

## Setup

```bash
pip install azure-appconfiguration-provider azure-identity
```

Endpoint pattern: `https://<store-name>.azconfig.io`

---

## Connect và Retrieve

```python
from azure.appconfiguration.provider import load
from azure.identity import DefaultAzureCredential
import os

endpoint = os.environ.get("AZURE_APPCONFIG_ENDPOINT")
credential = DefaultAzureCredential()

config = load(endpoint=endpoint, credential=credential)

# Dictionary syntax
model_endpoint = config["OpenAI:Endpoint"]
batch_size = config["Pipeline:BatchSize"]
```

`config` = Mapping object (Python dict-like). Default: load tất cả keys với **null label**.

**RBAC roles:**
- **`App Configuration Data Reader`** — read-only (app managed identities)
- **`App Configuration Data Owner`** — full CRUD (operators, CI/CD)

---

## SettingSelector — Filter Keys

```python
from azure.appconfiguration.provider import load, SettingSelector

selects = [
    SettingSelector(key_filter="Pipeline:*", label_filter="Production")
]

config = load(
    endpoint=endpoint,
    credential=DefaultAzureCredential(),
    selects=selects
)

batch_size = config["Pipeline:BatchSize"]
```

`key_filter` hỗ trợ wildcards (`*`). Multiple selectors: last selector wins cho cùng key.

---

## Trim Key Prefixes

```python
config = load(
    endpoint=endpoint,
    credential=DefaultAzureCredential(),
    trim_prefixes=["DocPipeline:"]
)

# Access "DocPipeline:OpenAI:Endpoint" as "OpenAI:Endpoint"
model_endpoint = config["OpenAI:Endpoint"]
```

Useful khi nhiều apps share cùng store với unique prefix per app.

---

## Dynamic Refresh

```python
from azure.appconfiguration.provider import load, WatchKey

config = load(
    endpoint=endpoint,
    credential=DefaultAzureCredential(),
    refresh_on=[WatchKey("Sentinel")],    # Sentinel key để signal refresh
    refresh_interval=60
)

# Trong request handler hoặc app loop
config.refresh()

batch_size = config["Pipeline:BatchSize"]    # Fresh value sau refresh
```

**Sentinel key pattern:** Operator update settings → update `Sentinel` key → trigger app reload. Tất cả changes load cùng lúc (không partial update).

`refresh()` phải được gọi **explicitly**. Nếu `refresh_interval` chưa elapsed → return immediately.

---

## Bản chất bài này là gì?

**Một câu:** App Configuration là centralized key-value store trên cloud để tách settings khỏi code, truy cập qua Python `load()` với dictionary syntax, có filter và dynamic refresh mà không cần restart app.

### So sánh với .env và K8s ConfigMap

| | .env / K8s ConfigMap | App Configuration |
|---|---|---|
| **Storage** | File trong repo hoặc K8s object | Cloud-hosted service, centralized |
| **Update** | Cần rebuild image / redeploy pod | Dynamic refresh — `config.refresh()` không restart |
| **Multi-env** | Separate file / separate ConfigMap | Labels trên cùng key (`null`, `Dev`, `Prod`) |
| **Filter khi load** | Không có, load tất cả | `SettingSelector(key_filter, label_filter)` + wildcards |
| **Hierarchical key** | `OPENAI_ENDPOINT=...` | `OpenAI:Endpoint` với `:` hoặc `/` |
| **Sensitive values** | Secrets trong K8s Secret / .env riêng | KV references trỏ sang Key Vault |

**`SettingSelector` multiple — last selector wins:** Load null label trước (defaults), load env label sau (overrides). Selector thứ hai ghi đè selector thứ nhất cho cùng key — không phải "first match wins."

**Dynamic refresh không tự động:** Provider là pull model. `config.refresh()` phải được gọi explicitly trong code. Nếu `refresh_interval` chưa elapsed → return immediately mà không gọi API. Sentinel pattern đảm bảo atomic reload.

---

## Checklist ghi nhớ cho AI-200

- [ ] App Configuration endpoint: `https://<name>.azconfig.io`
- [ ] Data model: Key (hierarchical `:` hoặc `/`) · Value (≤10KB) · Label · Content type
- [ ] Null label = default (no label assigned)
- [ ] `load()` → Mapping object, dictionary syntax
- [ ] **`App Configuration Data Reader`** = read-only cho app managed identity
- [ ] `SettingSelector(key_filter, label_filter)` · wildcards support
- [ ] Multiple selectors: **last selector wins** cho same key
- [ ] `trim_prefixes` → clean key names trong code
- [ ] Dynamic refresh: `WatchKey("Sentinel")` + `config.refresh()` explicitly

---

[← Mục lục](../README.md) · [🏠 Mục lục](../README.md) · [Bài 2 →](./appconfig-m1-bai2-labels-feature-flags.md)
