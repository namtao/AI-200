# Bài 1 — Store and Organize Secrets, Keys, and Certificates

> Khoá: AI-200 · Key Vault — Manage application secrets with Azure Key Vault

---

## Azure Key Vault là gì?

Cloud-hosted service cho centralized, secure storage của API keys, connection strings, encryption keys, TLS certificates. Authentication qua Microsoft Entra ID + Azure RBAC.

---

## 3 Object Types

| Type | Lưu gì | AI use case |
|---|---|---|
| **Secrets** | Arbitrary string ≤25 KB | API keys, connection strings, passwords, tokens |
| **Keys** | Cryptographic keys (RSA, EC, symmetric) | Encrypt training data, sign model artifacts. **Operations performed server-side** |
| **Certificates** | X.509 certs + private keys | HTTPS endpoints, mutual TLS |

**Secrets** phổ biến nhất cho AI apps. `content_type` property (≤255 chars) signal format (`text/plain`, `application/json`).

```python
client.set_secret(
    "openai-api-key",
    "sk-abc123def456",
    content_type="text/plain",
    tags={"environment": "production", "team": "ai-platform"}
)
```

---

## 2 Service Tiers

| Tier | Key protection | Khi dùng |
|---|---|---|
| **Standard** | Software (FIPS 140 Level 1) | Most scenarios |
| **Premium** | **HSM (FIPS 140-3 Level 3)** | Regulatory/compliance requirements |

---

## Tổ chức Vault

**Vault-per-application + environment:** `kv-ragpipeline-dev`, `kv-ragpipeline-staging`, `kv-ragpipeline-prod`. Limit blast radius, simplify RBAC.

**Vault naming:** Globally unique, 3-24 chars, alphanumeric + hyphens, start với letter, end letter/digit, không consecutive hyphens.

**Tags:** Tối đa 15 tags (name ≤512 chars, value ≤256 chars). Không lưu sensitive data trong tags.

---

## RBAC Roles

| Role | Permissions | Assign to |
|---|---|---|
| **Key Vault Secrets User** | **Read-only secret values** | **App managed identities** |
| Key Vault Secrets Officer | Full CRUD on secrets | Operators, CI/CD |
| Key Vault Administrator | All data plane ops | Admin only |
| Key Vault Reader | Metadata only (names, properties, no values) | Monitoring tools |

> **`Key Vault Contributor`** = **control plane only** (create/delete vault) — **KHÔNG** grant đọc secrets.

---

## Soft Delete và Purge Protection

- **Soft delete:** Default enabled, không thể disable. Deleted objects retained 7-90 days (default 90, **set at creation, cannot change**).
- **Purge protection:** Optional. Ngăn permanent deletion trong retention window. **Recommended cho production.**

Khi vault bị soft-delete: RBAC assignments và Event Grid subscriptions cũng bị delete → phải recreate thủ công khi recover.

---

## Bản chất bài này là gì?

**Một câu:** Key Vault là cloud HSM-backed secret store với 3 object types (Secrets/Keys/Certificates), truy cập qua Azure RBAC — và `Key Vault Contributor` là cái bẫy kinh điển vì nó chỉ manage vault resource, không đọc được secrets.

### So sánh với HashiCorp Vault và K8s Secrets

| | K8s Secrets | HashiCorp Vault | Azure Key Vault |
|---|---|---|---|
| **Encryption at rest** | base64 (không phải encrypt, exam trap!) | Có | Có — AES-256, HSM (Premium) |
| **HSM support** | Không | Có (Enterprise) | Có (Standard FIPS 140-1, Premium FIPS 140-3 Level 3) |
| **Audit log per access** | Không | Có | Có — Azure Monitor / Diagnostic logs |
| **Key operations server-side** | Không | Có | Có — key never leaves vault |
| **RBAC** | K8s RBAC | Vault policies | Azure RBAC (Entra ID) |
| **Soft delete** | Không | Không | Có — default enabled, 7-90 days |
| **Managed identity auth** | Service account | N/A | Có — DefaultAzureCredential |
| **Multi-tenant** | Cluster-scoped | Cluster/namespace | Azure subscription |

**`Key Vault Contributor` không đọc được secrets:** Đây là exam trap số 1. Role này là control plane (tạo/xóa vault). Data plane access cần `Key Vault Secrets User` (read-only) hoặc `Key Vault Secrets Officer` (CRUD). Tên "Contributor" gây nhầm lẫn.

**K8s Secrets không phải encrypted:** base64 encoding ≠ encryption. Nếu cần encryption at rest cho K8s Secrets, phải enable etcd encryption riêng. Azure Key Vault encrypt by default.

---

## Checklist ghi nhớ cho AI-200

- [ ] 3 types: **Secrets** (strings) · **Keys** (crypto, server-side ops) · **Certificates** (X.509)
- [ ] Secrets: up to **25 KB**, arbitrary strings
- [ ] Keys: vault performs operations — key material **never leaves vault**
- [ ] Standard (software) vs Premium (**HSM**)
- [ ] `Key Vault Contributor` = control plane ONLY, không đọc secrets
- [ ] **`Key Vault Secrets User`** = read-only, assign cho app managed identity
- [ ] Soft delete: default enabled · retention 7-90 days (set at creation, không đổi được)
- [ ] Purge protection: optional, **recommended production**

---

[← Assessment](./appconfig-m1-module-assessment.md) · [🏠 Mục lục](../README.md) · [Bài 2 →](./keyvault-m1-bai2-sdk.md)
