# Bài 2 — Retrieve Secrets Using Azure SDK Client Libraries

> Khoá: AI-200 · Key Vault — Manage application secrets with Azure Key Vault

---

## Setup

```bash
pip install azure-keyvault-secrets azure-identity
```

Vault URL pattern: `https://<vault-name>.vault.azure.net/`

---

## DefaultAzureCredential — Auth Chain

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

credential = DefaultAzureCredential()
client = SecretClient(
    vault_url="https://kv-ragpipeline-prod.vault.azure.net/",
    credential=credential
)
```

**Chain order (thử theo thứ tự):**
1. EnvironmentCredential (env vars `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_CLIENT_SECRET`)
2. WorkloadIdentityCredential (Kubernetes)
3. **ManagedIdentityCredential** ← Production (Azure compute)
4. **AzureCliCredential** ← Local dev (`az login`)
5. AzureDeveloperCliCredential (`azd auth login`)
6. AzurePowerShellCredential

Same code, no changes between local và production.

---

## Retrieve Secret

```python
secret = client.get_secret("openai-api-key")

api_key = secret.value                         # Secret string
version = secret.properties.version           # GUID
created = secret.properties.created_on
content_type = secret.properties.content_type
```

---

## List Secret Properties (không reveal values)

```python
secret_properties = client.list_properties_of_secrets()

for prop in secret_properties:
    print(f"Secret: {prop.name}, Enabled: {prop.enabled}, Tags: {prop.tags}")
```

Dùng khi: validate required secrets tồn tại trước khi app start, discovery, monitoring.

---

## Error Handling

```python
from azure.core.exceptions import ResourceNotFoundError, HttpResponseError, ServiceRequestError

def get_secret_safely(client, secret_name):
    try:
        return client.get_secret(secret_name).value
    except ResourceNotFoundError:
        # Secret không tồn tại — configuration error, needs operator
        raise
    except HttpResponseError as e:
        # Auth failure, authorization error, server error
        print(f"{e.status_code} - {e.message}")
        raise
    except ServiceRequestError:
        # Network issue — may be transient, can retry
        raise
```

| Exception | Nguyên nhân | Action |
|---|---|---|
| `ResourceNotFoundError` | Secret không tồn tại | Config error — cần operator |
| `HttpResponseError` | Auth/authorization/server | Check RBAC role |
| `ServiceRequestError` | Network issue | May retry |

---

## Async Client

```python
from azure.identity.aio import DefaultAzureCredential
from azure.keyvault.secrets.aio import SecretClient

async def get_ai_credentials():
    credential = DefaultAzureCredential()
    client = SecretClient(
        vault_url="https://kv-ragpipeline-prod.vault.azure.net/",
        credential=credential
    )

    async with client:
        secret = await client.get_secret("openai-api-key")

    await credential.close()
    return secret.value
```

Dùng cho high-throughput async services (FastAPI, aiohttp). Create client **once at startup**, reuse.

---

## Bản chất bài này là gì?

**Một câu:** `SecretClient` + `DefaultAzureCredential` là standard pattern để đọc secrets từ Key Vault, với `DefaultAzureCredential` tự động chain từ ManagedIdentity (production) sang AzureCLI (local dev) mà không cần thay đổi code.

### So sánh DefaultAzureCredential với các credential class khác

| | `ManagedIdentityCredential` | `EnvironmentCredential` | `DefaultAzureCredential` |
|---|---|---|---|
| **Production (Azure compute)** | Có | Có (nếu set env vars) | Có (ManagedIdentity trong chain) |
| **Local dev** | Không — fail | Chỉ nếu set `AZURE_CLIENT_SECRET` | Có (AzureCLI trong chain) |
| **Code thay đổi giữa env** | Cần — 2 code paths | Có thể — phụ thuộc env vars | Không — same code everywhere |
| **Kubernetes** | `WorkloadIdentityCredential` | Không | Có (WorkloadIdentity trong chain) |
| **Service principal** | Không | Có — env vars | Có (EnvironmentCredential đứng đầu chain) |

**`list_properties_of_secrets()` không trả về values:** Method này chỉ trả về metadata (tên, enabled, tags, expiry). Để đọc value phải gọi `get_secret(name)` riêng. Dùng để validate secrets tồn tại khi startup mà không phải fetch tất cả values.

**Async client cần import từ `.aio` submodule:** `from azure.keyvault.secrets.aio import SecretClient` và `from azure.identity.aio import DefaultAzureCredential` — không phải cùng package với sync client. Create once at startup, reuse — tạo mới mỗi request là anti-pattern.

---

## Checklist ghi nhớ cho AI-200

- [ ] `SecretClient` + `DefaultAzureCredential` = standard pattern
- [ ] Vault URL: `https://<name>.vault.azure.net/`
- [ ] DefaultAzureCredential: ManagedIdentity (prod) → AzureCLI (local dev)
- [ ] `secret.value` = secret string · `secret.properties.version` = GUID
- [ ] `list_properties_of_secrets()` = names + metadata, **không reveal values**
- [ ] `ResourceNotFoundError` = config error · `HttpResponseError` = auth/authz · `ServiceRequestError` = network
- [ ] Async: `azure.keyvault.secrets.aio` · `azure.identity.aio`
- [ ] Async client: create once at startup, reuse across requests

---

*Bài tiếp theo: Bài 3 — Handle secret versioning and rotation*
