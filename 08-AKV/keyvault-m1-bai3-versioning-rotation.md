# Bài 3 — Handle Secret Versioning and Rotation

> Khoá: AI-200 · Key Vault — Manage application secrets with Azure Key Vault

---

## Versioning

Mỗi lần `set_secret()` cùng tên → tạo **new version** (unique GUID). Old versions vẫn accessible.

```python
# Create initial version
initial = client.set_secret("cosmosdb-connection-string", "AccountEndpoint=https://old...")
print(f"Version 1: {initial.properties.version}")

# Create new version
updated = client.set_secret("cosmosdb-connection-string", "AccountEndpoint=https://new...")

# get_secret() không có version → luôn return LATEST ENABLED version
latest = client.get_secret("cosmosdb-connection-string")

# Retrieve specific version
specific = client.get_secret("cosmosdb-connection-string",
                              version=initial.properties.version)
```

---

## List All Versions

```python
versions = client.list_properties_of_secret_versions("cosmosdb-connection-string")

for version in versions:
    print(f"Version: {version.version}, Created: {version.created_on}, Enabled: {version.enabled}")
```

Sau rotation: disable old versions với `update_secret_properties(enabled=False)`.

---

## Rotation Strategies

### 1. Manual rotation

Operator/CI/CD tạo new secret version → restart app để pick up. Đơn giản nhưng có downtime window.

### 2. Automated rotation với Event Grid

```
SecretNearExpiry (30 days before expiry)
    → Event Grid
    → Azure Function
    → Generate new credential
    → set_secret() new version
```

`SecretExpired` event fires khi expiry date passes.

### 3. Dual-credential rotation (zero downtime)

Cho services hỗ trợ 2 active keys đồng thời (Azure Storage, Cosmos DB):

1. Generate new secondary key trên target service
2. `set_secret()` new version với new key
3. Wait cho tất cả instances pick up new version
4. Regenerate old primary key

→ **Luôn có ít nhất 1 valid credential** trong rotation window.

---

## Zero-downtime Rotation trong App Code

```python
def call_downstream_service(client, secret_name, max_retries=1):
    secret_value = client.get_secret(secret_name).value

    for attempt in range(max_retries + 1):
        try:
            return connect_to_service(secret_value)
        except AuthenticationError:
            if attempt < max_retries:
                # Credential rotated → refresh from Key Vault
                secret_value = client.get_secret(secret_name).value
            else:
                raise
```

Normal operation: dùng cached value. Auth fail → refetch từ KV (get latest version) → retry. Chỉ tốn 1 extra KV call khi rotation xảy ra.

---

## Expiration Dates

```python
from datetime import datetime, timezone, timedelta

expiration_date = datetime.now(timezone.utc) + timedelta(days=90)

client.set_secret(
    "openai-api-key",
    "sk-newkey789",
    expires_on=expiration_date,
    content_type="text/plain",
    tags={"rotation-policy": "90-days"}
)
```

Expired secrets **vẫn accessible** qua `get_secret()` — expiration là **informational**, không hard block. Kết hợp với Event Grid `SecretNearExpiry` để tự động rotate.

---

## Bản chất bài này là gì?

**Một câu:** Key Vault versioning là append-only — mỗi `set_secret()` tạo GUID mới, `get_secret()` không version trả về latest enabled, và dual-credential rotation là cách duy nhất đạt zero downtime cho services hỗ trợ 2 active keys.

### So sánh 3 rotation strategies

| | Manual Rotation | Automated (Event Grid) | Dual-credential |
|---|---|---|---|
| **Trigger** | Operator thủ công | `SecretNearExpiry` event (30 days trước) | Operator hoặc automated |
| **Downtime** | Có — cần restart app | Phụ thuộc service | Không — always ≥1 valid credential |
| **Zero downtime** | Không | Có thể (nếu dual-credential) | Có |
| **Complexity** | Thấp | Cao (Azure Function + Event Grid) | Trung bình |
| **Yêu cầu service** | Không | Không | Service phải support 2 active keys |
| **Ví dụ service** | Bất kỳ | Bất kỳ | Azure Storage, Cosmos DB |

**`get_secret()` không version → latest ENABLED version, không phải latest version:** Nếu latest version bị disable (enabled=False), get_secret() trả về version enabled gần nhất. Đây là cơ chế rollback — disable new version để tự động fall back về previous.

**`SecretNearExpiry` fires 30 days trước, `SecretExpired` fires khi expired — nhưng expired secret vẫn accessible:** Expiration date là informational, không hard-block. App vẫn đọc được expired secret nếu không có code check expiry.

---

## Checklist ghi nhớ cho AI-200

- [ ] Mỗi `set_secret()` cùng tên = **new version** (GUID)
- [ ] `get_secret()` không version → **latest enabled version**
- [ ] `get_secret(name, version=id)` → specific version
- [ ] 3 rotation strategies: Manual, Automated (Event Grid), Dual-credential
- [ ] **Dual-credential** = zero downtime cho services với 2 active keys
- [ ] `SecretNearExpiry` fires **30 days before** expiry · `SecretExpired` khi expired
- [ ] Expiration date = **informational** — không block access
- [ ] Zero-downtime: catch auth error → refetch latest → retry

---

*Bài tiếp theo: Bài 4 — Implement caching strategies*
