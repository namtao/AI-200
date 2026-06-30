# Bài 1 — Define ConfigMaps for Application Settings

> Khoá: AI-200 · AKS — Configure applications on Azure Kubernetes Service

---

## Tại sao cần ConfigMap?

Nếu hardcode config (endpoint URL, feature flag...) vào container image → mỗi lần thay đổi config phải rebuild image. ConfigMap giải quyết bằng cách **tách config ra khỏi image**, inject vào Pod lúc runtime.

**Nguyên tắc:** ConfigMap cho non-sensitive data. Secret cho sensitive data (bài sau).

---

## Định nghĩa ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-settings
data:
  FEATURE_X_ENABLED: "true"
  SERVICE_ENDPOINT: "https://api.example.com"
  # Multi-line value — mỗi key thành một file khi mount
  app.config: |
    log_level=info
    timeout_seconds=30
```

**Giới hạn:** Tối đa **1 MiB** per ConfigMap. Lớn hơn → dùng persistent volume hoặc external config service.

**Key naming:** Chỉ dùng alphanumeric, dashes, underscores, dots.

---

## Inject ConfigMap vào Pod — 2 cách

### Cách 1: Environment variables (từng key)

```yaml
containers:
- name: api
  image: myregistry.azurecr.io/web-api:v1
  env:
  - name: FEATURE_X_ENABLED
    valueFrom:
      configMapKeyRef:
        name: app-settings         # Tên ConfigMap
        key: FEATURE_X_ENABLED     # Key trong ConfigMap
  - name: SERVICE_ENDPOINT
    valueFrom:
      configMapKeyRef:
        name: app-settings
        key: SERVICE_ENDPOINT
```

### Cách 1b: Load tất cả keys cùng lúc (envFrom)

```yaml
containers:
- name: api
  envFrom:
  - configMapRef:
      name: app-settings    # Tất cả keys trong ConfigMap → env vars
```

Tiện hơn khi có nhiều config values, không cần liệt kê từng key.

### Cách 2: Mount như file (volume)

Dùng khi app đọc config từ file trên disk thay vì env var.

```yaml
spec:
  volumes:
  - name: config-volume
    configMap:
      name: app-settings
      # Optional: chỉ mount một số key nhất định
      # items:
      # - key: app.config
      #   path: application.conf
  containers:
  - name: api
    volumeMounts:
    - name: config-volume
      mountPath: /app/config    # Mỗi key → một file trong thư mục này
      readOnly: true
```

Kết quả: `/app/config/FEATURE_X_ENABLED`, `/app/config/SERVICE_ENDPOINT`, `/app/config/app.config`

**Khác biệt quan trọng giữa 2 cách:**

| | Env var | Volume mount |
|---|---|---|
| Auto-update khi ConfigMap thay đổi | ❌ Cần restart Pod | ✅ Kubernetes tự sync |
| Phù hợp với | App đọc từ `os.environ` | App đọc từ file config |
| subPath mount | N/A | ❌ Không auto-update nếu dùng subPath |

---

## Immutable ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-settings-v2
data:
  FEATURE_X_ENABLED: "true"
immutable: true
```

Khi set `immutable: true`:
- **Không thể sửa** data hoặc revert immutable
- **Xoá và tạo lại** để thay đổi
- **Lợi ích performance:** Kubernetes đóng watch trên resource này → giảm tải API server trong cluster lớn có nhiều ConfigMap

**Dùng khi:** Config gắn với một app version cụ thể, thay đổi đồng nghĩa với redeploy.

---

## Tích hợp với Azure App Configuration

**Azure App Configuration** là dịch vụ quản lý config tập trung cho nhiều app và environment. **Azure App Configuration Kubernetes Provider** chạy trong cluster và tự động generate ConfigMap từ data trong App Configuration.

Lợi ích:
- Manage feature flags và config từ một chỗ cho nhiều AKS cluster
- Sync thay đổi từ App Configuration → cluster tự động
- Hỗ trợ Key Vault references trong App Configuration

---

## Verify bằng kubectl

```bash
kubectl apply -f configmap.yaml
kubectl apply -f deployment.yaml

# Xem nội dung ConfigMap
kubectl describe configmap app-settings

# Kiểm tra Pod đã nhận config chưa
kubectl exec <pod-name> -- printenv | grep FEATURE
```

---

## Bản chất bài này là gì?

**Một câu:** ConfigMap tách config ra khỏi container image để cùng một image chạy với nhiều config khác nhau — triết lý 12-factor app, tương tự env file hoặc config map trong Docker Compose nhưng Kubernetes-native và có thêm auto-sync.

### So sánh với Docker và ACA

| Cách inject config | Docker | ACA | AKS ConfigMap |
|---|---|---|---|
| Từng biến | `docker run -e KEY=VAL` | `--env-vars KEY=VAL` | `configMapKeyRef` |
| Tất cả biến | `docker run --env-file .env` | N/A | `envFrom: configMapRef` |
| File config | `-v ./config:/app/config` | N/A | Volume mount ConfigMap |
| Auto-update khi config thay đổi | ❌ | ❌ | ✅ (chỉ volume mount, không phải env var) |

**Điểm đặc trưng của ConfigMap so với Docker:**
- **envFrom** (load tất cả key) = gần với `--env-file` nhất, tiện hơn liệt kê từng key
- **Volume mount auto-sync** = khi update ConfigMap, file trong container tự cập nhật (không cần restart Pod) — không có trong Docker
- **`subPath` mount không auto-sync** — đây là exception cần nhớ

**Env var vs volume mount — quy tắc đơn giản:**
- App đọc từ `os.environ` hoặc `process.env` → dùng env var
- App đọc từ file trên disk → dùng volume mount

---

## Checklist ghi nhớ cho AI-200

- [ ] **ConfigMap** = non-sensitive config · **Secret** = sensitive data
- [ ] Giới hạn **1 MiB** per ConfigMap
- [ ] Key naming: alphanumeric, dashes, underscores, dots
- [ ] `configMapKeyRef` → inject từng key · `configMapRef` (envFrom) → inject tất cả
- [ ] Volume mount: mỗi key → một file trong thư mục mount
- [ ] **Env var** không auto-update khi ConfigMap thay đổi → cần restart Pod
- [ ] **Volume mount** tự sync khi ConfigMap update (trừ subPath)
- [ ] `immutable: true` → không thể sửa, phải delete + recreate
- [ ] Verify: `kubectl exec <pod> -- printenv | grep <KEY>`

---

[← Assessment](./aks-m1-module-assessment.md) · [🏠 Mục lục](../README.md) · [Bài 2 →](./aks-m2-bai2-secrets.md)
