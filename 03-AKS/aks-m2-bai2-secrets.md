# Bài 2 — Implement Secrets for Sensitive Data

> Khoá: AI-200 · AKS — Configure applications on Azure Kubernetes Service

---

## Tại sao cần Secret?

AI application thường gọi external service — model endpoint, database, vector store — và cần API key, connection string để authenticate. **Kubernetes Secret** lưu sensitive value này:
- Tách khỏi source code và container image
- Không bị commit lên Git
- Inject vào Pod lúc runtime, không ghi ra disk theo default

---

## Tạo Secret

### Bằng kubectl (trực tiếp, nhanh)

```bash
kubectl create secret generic app-secrets \
  --from-literal=DB_CONNECTION="Host=db;User=app;Password=secure" \
  --from-literal=API_KEY="your-api-key"
```

**Secret types phổ biến:**
- `generic` (opaque) — API key, password, connection string
- `kubernetes.io/dockerconfigjson` — Container registry credentials
- `kubernetes.io/tls` — TLS certificate và key

### Bằng YAML manifest

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  API_KEY: eW91ci1hcGkta2V5    # Base64 encoded
```

> **Không commit YAML manifest chứa secret value lên Git** — dù base64 trông khó đọc, nó không phải encryption, chỉ là encoding.

---

## Reference Secret trong Deployment

```yaml
containers:
- name: api
  image: myregistry.azurecr.io/web-api:v1
  env:
  - name: DB_CONNECTION
    valueFrom:
      secretKeyRef:
        name: app-secrets        # Tên Secret object
        key: DB_CONNECTION       # Key trong Secret
  - name: API_KEY
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: API_KEY
```

Kubernetes inject giá trị thực của secret vào env var. Value nằm trong memory, không ghi ra disk theo default.

---

## Verify bằng kubectl

```bash
# Liệt kê secrets (không thấy value)
kubectl get secrets

# Xem metadata và keys — không expose value
kubectl describe secret app-secrets

# Verify deployment reference đúng không
kubectl describe deployment web-api
```

`kubectl describe secret` hiển thị key names và size, **không bao giờ hiển thị value**. Đây là behavior bảo mật của Kubernetes.

---

## Tích hợp với Azure cho Production

### Option 1: Azure Key Vault + Secrets Store CSI Driver

CSI Driver mount secret từ Key Vault **trực tiếp vào Pod filesystem như file**. Secret không đi qua Kubernetes etcd.

- Secret ở lại trong Key Vault
- Sync khi Pod start, cập nhật khi Key Vault thay đổi (không cần restart)
- Audit trail chi tiết cho mỗi lần access
- Hỗ trợ automated rotation

**Dùng khi:** Cần strict compliance controls, audit per-access, muốn secret không bao giờ rời Key Vault.

### Option 2: Azure App Configuration + Key Vault references

App Configuration Kubernetes Provider đọc Key Vault references từ App Configuration, fetch actual values từ Key Vault, rồi generate Kubernetes Secret resources. App nhận secret như standard Kubernetes Secret — không cần authenticate trực tiếp với Key Vault.

**Dùng khi:** Muốn quản lý secret mappings tập trung, cung cấp secret cho app như Kubernetes resource thông thường.

### So sánh

| | Direct Key Vault (CSI) | App Config + Key Vault |
|---|---|---|
| Audit per-access | ✅ Fine-grained | ⚠️ Less granular |
| Secret rời Key Vault | ❌ Không | ✅ Được generate thành K8s Secret |
| Quản lý mapping tập trung | ❌ | ✅ App Configuration |
| App cần biết Key Vault | ❌ (CSI handle) | ❌ (Provider handle) |

---

## Best Practices

- **RBAC:** Chỉ grant Secret view/modify cho service account và user thực sự cần
- **Không commit manifest với literal value** — dùng `kubectl create secret` hoặc external secret manager
- **Rotate định kỳ** — update secret rồi trigger Deployment rollout để Pod nhận value mới
- **Key Vault cho production** — automated rotation, audit trail, CSI driver sync không cần restart
- **etcd encryption** — enable khi tạo cluster để thêm lớp bảo vệ

---

## Bản chất bài này là gì?

**Một câu:** Kubernetes Secret = Kubernetes-native credential store với base64 encoding (không phải encryption), và production setup thêm Azure Key Vault để có audit trail và auto-rotation.

### So sánh với các giải pháp khác

| Giải pháp | Lưu ở đâu | Encryption | Auto-rotate | Audit |
|---|---|---|---|---|
| Docker env var | Image/compose file | ❌ Plain text | ❌ | ❌ |
| Docker secret (Swarm) | Swarm manager | ✅ Encrypted at rest | ❌ | ❌ |
| ACA Secret | ACA config | ❌ base64 | ❌ | ❌ |
| K8s Secret (native) | etcd | ❌ base64 (hoặc encrypted etcd) | ❌ | ❌ |
| K8s Secret + CSI Driver + Key Vault | Key Vault | ✅ | ✅ | ✅ Fine-grained |

**Base64 không phải encryption — đây là exam trap:** `kubectl describe secret` ẩn value nhưng `kubectl get secret -o yaml` vẫn expose value dưới dạng base64. Bất kỳ ai có `kubectl get secret` permission đều đọc được. Đó là lý do phải dùng RBAC chặt và Key Vault cho production.

**CSI Driver + Key Vault = "secret không rời Key Vault"**: Secret không bao giờ được lưu trong K8s etcd — CSI Driver mount thẳng từ Key Vault vào Pod filesystem. Khác với cách tạo K8s Secret từ Key Vault value (value đi qua etcd).

---

## Checklist ghi nhớ cho AI-200

- [ ] Secret cho: API key, password, connection string, TLS cert, registry credential
- [ ] Secret types: `generic` (opaque), `dockerconfigjson`, `tls`
- [ ] `kubectl create secret generic <name> --from-literal=KEY=VALUE`
- [ ] `secretKeyRef` trong Deployment để reference secret value
- [ ] `kubectl describe secret` hiển thị key names nhưng **không expose value**
- [ ] **CSI Driver + Key Vault** → secret ở lại Key Vault, mount như file, auto-update
- [ ] **App Config + Key Vault** → Provider generate K8s Secret từ Key Vault references
- [ ] Không bao giờ commit YAML manifest có secret value lên Git

---

[← Bài 1](./aks-m2-bai1-configmaps.md) · [🏠 Mục lục](../README.md) · [Bài 3 →](./aks-m2-bai3-persistent-storage.md)
