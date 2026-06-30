# Bài 4 — Manage Secrets and Configuration in Functions

> Khoá: AI-200 · Azure Functions — Build serverless AI backends with Azure Functions

---

## Application Settings

Key-value pairs expose đến function code dưới dạng **environment variables**. Azure encrypt at rest, decrypt khi inject vào process memory.

```python
import os

ai_endpoint = os.environ["AI_SERVICE_ENDPOINT"]       # Required
model_id = os.environ["MODEL_ID"]
confidence_threshold = float(os.getenv("CONFIDENCE_THRESHOLD", "0.85"))  # With default
```

Local: `local.settings.json` Values object. Production: Azure-managed app settings.

---

## Key Vault References

Lưu sensitive values trong Azure Key Vault, reference từ application settings. Runtime resolve tại app startup.

**Syntax options:**

```
# URI syntax (versionless — auto-pick latest)
@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/AiServiceKey/)

# Named syntax
@Microsoft.KeyVault(VaultName=myvault;SecretName=AiServiceKey)
```

Function code đọc bình thường qua `os.environ`:
```python
api_key = os.environ["AI_SERVICE_KEY"]   # Resolved từ Key Vault transparently
```

**Requirements:** Managed identity với **`Key Vault Secrets User` role** trên vault.

### Versionless vs Versioned

| | Versionless URI | Versioned URI |
|---|---|---|
| Rotation | Auto-detect new version trong **24 giờ** | Pinned đến version cụ thể |
| Dùng khi | **Scheduled rotation** (AI API keys) | Controlled deployment |

**Config change → immediate refetch** của tất cả Key Vault references.

---

## Azure App Configuration

Centralized store cho non-secret config values và feature flags. Valuable khi **nhiều function apps / microservices chia sẻ config**.

```python
from azure.appconfiguration.provider import load
from azure.identity import DefaultAzureCredential
import os

config = load(
    endpoint=os.environ["APP_CONFIG_ENDPOINT"],
    credential=DefaultAzureCredential(),
    selects=[{"key_filter": "AIBackend/*"}]
)

model_endpoint = config["AIBackend/ModelEndpoint"]
inference_timeout = int(config["AIBackend/InferenceTimeout"])
```

**Label-based filtering**: Load different config sets cho different environments (dev, staging, prod) từ same store.

**Combined pattern:** App Configuration cho non-sensitive values + Key Vault cho secrets. App Configuration hỗ trợ Key Vault references → unified configuration surface.

---

## Local Development Config

```json
// local.settings.json
{
    "IsEncrypted": false,
    "Values": {
        "AzureWebJobsStorage": "UseDevelopmentStorage=true",
        "FUNCTIONS_WORKER_RUNTIME": "python",
        "AI_SERVICE_ENDPOINT": "https://local-or-real-endpoint",
        "COSMOS_CONNECTION": "AccountEndpoint=https://localhost:8081/;..."
    }
}
```

```bash
# Download từ deployed app
func azure functionapp fetch-app-settings <app-name>

# Encrypt để bảo vệ secrets on dev machine
func settings encrypt
```

**Best practice:** Local settings point đến local services (Azurite, Cosmos emulator, local Redis). Production deploy → swap sang real endpoints hoặc identity-based connections.

---

## Bản chất bài này là gì?

**Một câu:** Secrets management = 3 tầng: app settings (env vars, encrypted at rest) → Key Vault references (managed identity, versionless URI cho auto-rotation) → App Configuration (non-secret centralized config cho multi-service); không bao giờ hardcode secrets.

### So sánh với AWS Secrets Manager / HashiCorp Vault / Google Secret Manager

| | Azure Key Vault References | AWS Secrets Manager | HashiCorp Vault | Google Secret Manager |
|---|---|---|---|---|
| Integration | App setting reference syntax | Environment injection (Lambda) | Agent / SDK | SDK / Secret injection |
| Auto-rotation | Versionless URI → 24h pickup | Lambda rotation function | Lease renewal | Không native |
| Auth | Managed identity + KV Secrets User | IAM role | AppRole / JWT | Service account |
| Code change | Không (transparent via os.environ) | Có (SDK call) | Có (SDK call) | Có (SDK call) |
| Non-secret config | App Configuration (separate service) | Parameter Store | KV secrets | Không (separate) |
| Multi-env | App Configuration labels | Per-env secrets | Namespaces | Không native |
| Centralized multi-service | App Configuration | Parameter Store | KV | Secret Manager |

**Versionless vs Versioned URI:** Versionless (`/secrets/AiServiceKey/` không có GUID) → runtime tự detect version mới trong 24h. Versioned → phải update app setting khi rotate = effective redeploy. Production AI keys thường cần rotation → versionless là đúng.

**Exam trap:** Versionless URI auto-rotate trong **24 giờ** — không phải ngay lập tức. Nếu cần rotate ngay: config change trigger immediate refetch của tất cả KV references. Kết hợp: rotate key → touch any app setting → references reload ngay.

---

## Checklist ghi nhớ cho AI-200

- [ ] App settings = environment variables trong function code
- [ ] `os.environ["KEY"]` required · `os.getenv("KEY", "default")` optional với fallback
- [ ] Key Vault references: `@Microsoft.KeyVault(VaultName=..;SecretName=..)`
- [ ] Managed identity cần **`Key Vault Secrets User`** role
- [ ] **Versionless URI** → auto-rotate trong 24h · Versioned URI → pinned
- [ ] App Configuration: centralized non-secret config · labels cho multi-env
- [ ] `func settings encrypt` → protect secrets on dev machine
- [ ] Local: point đến local services · Production: real endpoints/identity-based

---

[← Bài 3](./backend-m3-bai3-triggers-bindings.md) · [🏠 Mục lục](../README.md) · [Bài 5 →](./backend-m3-bai5-identity-access.md)
