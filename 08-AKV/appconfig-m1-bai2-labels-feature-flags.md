# Bài 2 — Organize Settings with Labels and Feature Flags

> Khoá: AI-200 · App Configuration — Manage application settings with Azure App Configuration

---

## Labels cho Environment Differentiation

Cùng key, khác label → khác value. App code reference key name không đổi.

| Key | Label | Value |
|---|---|---|
| `Pipeline:BatchSize` | (null) | 10 (default) |
| `Pipeline:BatchSize` | Development | 10 |
| `Pipeline:BatchSize` | Staging | 50 |
| `Pipeline:BatchSize` | **Production** | **200** |

Code luôn: `config["Pipeline:BatchSize"]` — không cần conditional logic.

---

## Composition Pattern (Stacking)

```python
from azure.appconfiguration.provider import load, SettingSelector
import os

environment = os.environ.get("APP_ENVIRONMENT", "Development")

selects = [
    SettingSelector(key_filter="*", label_filter="\0"),          # Null label = defaults (load first)
    SettingSelector(key_filter="*", label_filter=environment)    # Env-specific overrides (load second)
]

config = load(
    endpoint=endpoint,
    credential=DefaultAzureCredential(),
    selects=selects
)

# Env-specific value nếu có, fallback đến default
batch_size = config["Pipeline:BatchSize"]
```

**Null label** = `"\0"` trong provider API. Load defaults trước, env-specific sau → **last selector wins**.

---

## Feature Flags

Special key-value pairs managed qua dedicated interface. Store với prefix `.appconfig.featureflag/` và content type `application/vnd.microsoft.appconfig.ff+json`. Provider abstract hết details.

**Use cases:**
- Progressive model rollouts (internal test trước production)
- Kill switches (emergency disable processing step)
- A/B testing configurations
- Staged pipeline activation

### Evaluate Feature Flags

```bash
pip install featuremanagement
```

```python
from azure.appconfiguration.provider import load
from azure.identity import DefaultAzureCredential
from featuremanagement import FeatureManager

config = load(
    endpoint=endpoint,
    credential=DefaultAzureCredential(),
    feature_flags_enabled=True    # Enable feature flag loading
)

feature_manager = FeatureManager(config)

if feature_manager.is_enabled("UseNewEmbeddingsModel"):
    process_with_new_model(document)
else:
    process_with_current_model(document)
```

`is_enabled()` evaluates flag **each time called** → works với dynamic refresh.

---

## Dynamic Refresh cho Feature Flags

```python
config = load(
    endpoint=endpoint,
    credential=DefaultAzureCredential(),
    feature_flags_enabled=True,
    feature_flag_refresh_enabled=True,    # Enable FF refresh independently
    refresh_on=[WatchKey("Sentinel")],
    refresh_interval=30
)

feature_manager = FeatureManager(config)

# In request handler
config.refresh()

if feature_manager.is_enabled("UseNewEmbeddingsModel"):
    process_with_new_model(document)
```

**Feature flag refresh và config refresh là độc lập:**
- FF change không trigger config settings refresh
- Config change không trigger FF refresh
- Cả hai refresh khi `config.refresh()` được gọi

---

## Key Naming Convention

```
OpenAI:Endpoint
OpenAI:DeploymentName
OpenAI:MaxTokens
CosmosDB:DatabaseName
CosmosDB:ContainerName
Pipeline:BatchSize
Pipeline:RetryCount
Pipeline:TimeoutSeconds
```

Hierarchical với `:` delimiter → filter bằng `key_filter="OpenAI:*"` để load chỉ OpenAI settings.

---

## Bản chất bài này là gì?

**Một câu:** Labels biến cùng một key thành nhiều variants per environment (composition pattern: load defaults → load overrides), còn Feature Flags là special key-value pairs cho phép toggle code path tại runtime mà không redeploy.

### So sánh với .env và K8s ConfigMap / environment variables

| | .env per environment | K8s ConfigMap per env | App Config Labels |
|---|---|---|---|
| **Multi-env** | `.env.dev`, `.env.prod` — separate files | Separate ConfigMap per namespace | Một store, cùng key, khác label |
| **Override logic** | Manual merge khi deploy | Mount đúng ConfigMap | `SettingSelector` stacking, last wins |
| **Feature flags** | Không có — dùng env var `FF_NEW_MODEL=true` | Không có — dùng ConfigMap value | Dedicated FF interface + `FeatureManager.is_enabled()` |
| **Toggle không redeploy** | Không — cần rebuild/restart | Không — cần update ConfigMap + restart | Có — dynamic refresh + `is_enabled()` eval mỗi call |
| **Null label** | N/A | N/A | `"\0"` trong API — represents default (no label) |

**Null label trong code là `"\0"` không phải `None`:** Khi dùng `SettingSelector` với `label_filter="\0"` thì match keys không có label. Đây là exam trap — gõ `None` sẽ không filter đúng.

**FF refresh độc lập từ config refresh:** `feature_flag_refresh_enabled=True` phải set riêng. Nhưng cả hai được triggered bởi cùng `config.refresh()` call — không có hai method khác nhau.

---

## Checklist ghi nhớ cho AI-200

- [ ] Labels: variants của cùng key cho khác environments
- [ ] Null label = `"\0"` trong provider · serves as default
- [ ] Composition: load null (defaults) → load env label (overrides) · **last wins**
- [ ] Feature flags: `feature_flags_enabled=True` + `pip install featuremanagement`
- [ ] `FeatureManager(config).is_enabled("FlagName")` → boolean
- [ ] Feature flag refresh: `feature_flag_refresh_enabled=True` (independent từ config refresh)
- [ ] Cả FF và config refresh khi `config.refresh()` called
- [ ] Use cases: progressive rollout, kill switch, A/B testing, staged activation

---

*Bài tiếp theo: Bài 3 — Reference Key Vault secrets from App Configuration*
