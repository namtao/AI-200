# Bài 4 — Decide What to Store in App Configuration vs Key Vault

> Khoá: AI-200 · App Configuration — Manage application settings with Azure App Configuration

---

## Design Intent

| Service | Built for | Capabilities |
|---|---|---|
| **App Configuration** | Non-sensitive settings | Labels, feature flags, snapshots, dynamic refresh, high throughput |
| **Key Vault** | Sensitive credentials | HSM encryption, audit logs per access, expiration, rotation, soft delete |

---

## Classification

**→ App Configuration (non-sensitive, không grant access):**
- Model endpoint URLs (`https://my-openai.openai.azure.com/`)
- Model deployment names (`gpt-4o`, `text-embedding-ada-002`)
- Embedding dimensions (1536, 3072)
- Batch sizes, retry counts, timeout values
- Feature flags (toggle processing paths)
- Logging levels, queue names, container names, database names
- Pipeline routing rules

**→ Key Vault (sensitive, grant access to resources):**
- Azure OpenAI API keys
- Database connection strings với embedded credentials
- Storage account access keys
- Third-party service tokens
- TLS certificates + private keys
- Encryption keys
- Passwords, service principal secrets

**Criterion:** Value này có thể cho phép someone access a resource không? Yes → Key Vault. No → App Configuration.

---

## Complementary Architecture

```
Application
    → load() một lần
    → App Configuration
        ├── Regular settings (direct values)
        └── KV References → Key Vault (secrets)
    → Unified dictionary
```

**Benefits:**
- Operators quản lý toàn bộ config surface ở một nơi (App Configuration)
- Labels, feature flags áp dụng cho cả regular settings và KV references
- App code đơn giản — same dictionary interface

---

## Rotation và Audit

| Requirement | Service |
|---|---|
| Rotation với audit trail, expiration | Key Vault |
| Operational changes (batch size, model version) | App Configuration |

---

## Anti-patterns cần tránh

| Anti-pattern | Vấn đề |
|---|---|
| **Secrets trong App Config trực tiếp** | Không có per-object audit, không HSM, không expiration/rotation |
| **Non-sensitive settings trong Key Vault** | Không có labels/feature flags, throttling limit thấp hơn, wasted security overhead |
| **Duplicate cùng value ở cả hai services** | Values có thể drift apart (sync issues) — chỉ dùng KV references |
| **Hard-code settings trong app code** | Mất khả năng update mà không redeploy |

---

## Summary Table

| Setting | Service | Ví dụ |
|---|---|---|
| Nonsensitive config | App Config (direct) | `OpenAI:Endpoint`, `Pipeline:BatchSize` |
| Feature toggles | App Config (feature flag) | `UseNewEmbeddingsModel` |
| API keys, tokens | Key Vault (via App Config reference) | `OpenAI:ApiKey` |
| Connection strings | Key Vault (via App Config reference) | `CosmosDB:ConnectionString` |
| TLS certificates | Key Vault (direct access) | Certificates for HTTPS |

---

## Bản chất bài này là gì?

**Một câu:** Decision rule chỉ có một câu hỏi — "value này có thể dùng để truy cập resource không?" — Yes → Key Vault, No → App Configuration; và hai service phải dùng complementary, không thay thế nhau.

### So sánh App Configuration vs Key Vault — khi nào dùng cái gì

| Criterion | App Configuration | Key Vault |
|---|---|---|
| **Sensitivity** | Non-sensitive — biết value không grant access | Sensitive — value alone grants resource access |
| **Ví dụ** | Endpoint URL, deployment name, batch size, queue name | API key, connection string với password, storage key |
| **Feature flags** | Có | Không |
| **Labels per env** | Có | Không (separate vault per env) |
| **Throttle limit** | Cao hơn | 4,000 GET / vault / 10s |
| **Audit per secret** | Không | Có — mỗi lần read được log |
| **HSM encryption** | Không | Có (Premium tier) |
| **Expiration/rotation** | Không có built-in | Có — expiry date, Event Grid rotation |
| **Anti-pattern** | Lưu secret trực tiếp (no HSM, no audit) | Lưu non-sensitive (no labels, wasted overhead) |

**Model endpoint URL → App Config, API key → Key Vault:** Đây là exam case điển hình. `gpt-4o` deployment name hoặc `https://my-openai.openai.azure.com/` không grant access — App Config. `sk-abc123` API key grant access — Key Vault.

**Duplicate value ở cả hai services = anti-pattern:** Nếu cần reference secret từ App Config context, dùng KV reference (pointer), không copy value. Copy = drift risk khi rotate.

---

## Checklist ghi nhớ cho AI-200

- [ ] App Config = non-sensitive (behavior settings) · Key Vault = sensitive (access grants)
- [ ] **Model endpoint URL** → App Config · **API key** → Key Vault
- [ ] Criterion: "can this value alone grant access to a resource?"
- [ ] Recommended: App Config as single entry point, KV references for secrets
- [ ] Secrets in App Config directly = anti-pattern (no HSM, no per-object audit)
- [ ] Non-sensitive in Key Vault = anti-pattern (no labels, lower throttle limits)
- [ ] Duplicate values = anti-pattern (drift risk) → use KV references instead
- [ ] Rotation + audit → Key Vault · Operational changes → App Configuration

---

*Module Assessment tiếp theo*
