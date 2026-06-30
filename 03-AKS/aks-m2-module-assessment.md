# Module Assessment — Configure Applications on Azure Kubernetes Service

> Khoá: AI-200 · AKS — Configure applications on Azure Kubernetes Service

---

## Câu 1

**You need to store a database connection string for your application running on AKS. The connection string contains a password and shouldn't be visible in your source code repository. Which Kubernetes resource should you use?**

- ConfigMap, because it stores configuration data
- Secret, because it stores sensitive values and keeps credentials out of source control ✅
- PersistentVolumeClaim, because it provides storage for application data

**Đáp án: Secret**

Secret là Kubernetes resource được thiết kế đúng cho sensitive values như connection string có password:
- Không commit vào source control
- Inject vào Pod lúc runtime qua `secretKeyRef`
- `kubectl describe secret` không expose value thực

```bash
kubectl create secret generic app-secrets \
  --from-literal=DB_CONNECTION="Host=db;User=app;Password=secure"
```

```yaml
env:
- name: DB_CONNECTION
  valueFrom:
    secretKeyRef:
      name: app-secrets
      key: DB_CONNECTION
```

Tại sao các đáp án còn lại sai:
- **ConfigMap:** Dùng cho non-sensitive config. Connection string có password là sensitive → Secret. ConfigMap data có thể bị thấy dễ dàng qua `kubectl describe configmap`.
- **PVC:** Cung cấp persistent file storage, không phải nơi lưu credential. PVC là cho file data như embeddings hay uploads, không phải cho config values.

---

## Câu 2

**Your AI application reads feature flags and service endpoints from environment variables. You want to update these settings without rebuilding your container image. How should you inject these nonsensitive values into your Pods?**

- Create a ConfigMap with the settings and reference the keys using configMapKeyRef in the Deployment ✅
- Store the values in a Secret and mount it as a volume
- Hardcode the values in the Deployment manifest and update the manifest when settings change

**Đáp án: ConfigMap với configMapKeyRef**

ConfigMap là đúng vì:
- Feature flags và endpoint URL là non-sensitive → ConfigMap (không cần Secret)
- Tách config khỏi image → update ConfigMap mà không rebuild image

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-settings
data:
  FEATURE_X_ENABLED: "true"
  SERVICE_ENDPOINT: "https://api.example.com"

# Deployment
env:
- name: FEATURE_X_ENABLED
  valueFrom:
    configMapKeyRef:
      name: app-settings
      key: FEATURE_X_ENABLED
```

Tại sao các đáp án còn lại sai:
- **Secret mounted as volume:** Secret là cho sensitive data — feature flags và endpoints không phải sensitive. Dùng Secret cho non-sensitive data là over-engineering và làm phức tạp quản lý không cần thiết.
- **Hardcode trong manifest:** Hardcode trong Deployment manifest vẫn yêu cầu `kubectl apply` mỗi khi thay đổi, và giá trị nằm trong manifest file (thường trong source control). Không tốt hơn hardcode trong image là bao.

---

## Câu 3

**You create a PersistentVolumeClaim in your AKS cluster. What happens when you apply the PVC manifest?**

- You must manually create an Azure Disk in the Azure portal before the PVC can bind
- AKS uses the specified StorageClass to automatically provision Azure storage that backs the PVC ✅
- The PVC remains unbound until you create a matching PersistentVolume manifest

**Đáp án: AKS tự động provision Azure storage**

AKS có built-in StorageClasses với **dynamic provisioning** — khi apply PVC, Kubernetes dùng StorageClass để tự động tạo Azure storage (Azure Disk hoặc Azure Files) phù hợp. Không cần tay hay tạo PV riêng.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: managed-csi    # AKS provision Azure Disk tự động
```

Tại sao các đáp án còn lại sai:
- **Phải tạo Azure Disk thủ công:** Đây là cách dùng **static provisioning** (tạo PV trước, PVC bind vào PV đó). Với dynamic provisioning qua StorageClass, không cần tạo thủ công. AKS mặc định dùng dynamic provisioning.
- **PVC unbound cho đến khi tạo PV:** Tương tự — đây là static provisioning. Với dynamic StorageClass, Kubernetes tự tạo PV phù hợp ngay khi PVC được apply.

---

## Câu 4

**Your application needs to access API keys stored in a Kubernetes Secret. You want to make the keys available as environment variables in the container. Which field should you use in the Deployment manifest to reference the Secret?**

- `valueFrom` with `secretKeyRef` ✅
- Volumes with secret type
- `configMapKeyRef` pointing to the Secret name

**Đáp án: `valueFrom` với `secretKeyRef`**

```yaml
env:
- name: API_KEY
  valueFrom:
    secretKeyRef:
      name: app-secrets      # Tên Secret object
      key: API_KEY           # Key trong Secret
```

`secretKeyRef` inject giá trị của key từ Secret vào environment variable. Value nằm trong memory container, không ghi ra disk.

Tại sao các đáp án còn lại sai:
- **Volumes with secret type:** Mount Secret như file, không như env var. File sẽ xuất hiện trên filesystem của container. Câu hỏi yêu cầu **environment variable**, không phải file.
- **`configMapKeyRef` pointing to Secret name:** `configMapKeyRef` chỉ đọc từ **ConfigMap**, không đọc từ Secret. Dùng sai field này sẽ gây error khi Pod start vì Kubernetes không tìm thấy Secret trong ConfigMap store.

---

## Câu 5

**You need to decide between mounting a ConfigMap as environment variables or as files. Your application reads a JSON configuration file at startup. Which approach should you choose?**

- Mount the ConfigMap as environment variables because all applications can read environment variables
- Mount the ConfigMap as files using a volume so the JSON file appears on disk where the application expects it ✅
- Store the JSON content in a Secret and use secretKeyRef to inject it as an environment variable

**Đáp án: Mount ConfigMap như file**

App đọc JSON config file từ disk → mount như volume là đúng. Mỗi key trong ConfigMap trở thành file trong thư mục được mount.

```yaml
# ConfigMap
data:
  config.json: |
    {
      "log_level": "info",
      "timeout": 30
    }

# Deployment
volumes:
- name: config-volume
  configMap:
    name: app-settings
containers:
- name: api
  volumeMounts:
  - name: config-volume
    mountPath: /app/config    # app đọc /app/config/config.json
    readOnly: true
```

Tại sao các đáp án còn lại sai:
- **Env vars:** JSON config không phù hợp để inject qua env var — JSON thường multi-line, có ký tự đặc biệt, và app được code để đọc từ file path cụ thể, không phải env var. Câu hỏi nói rõ "reads a JSON configuration file".
- **Secret + secretKeyRef:** JSON config file là non-sensitive (feature flags, timeouts...) → không cần Secret. `secretKeyRef` inject value như env var (không phải file), và Secret là cho sensitive data.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Resource cho sensitive data (connection string) | Secret |
| 2 | Inject non-sensitive config mà không rebuild image | ConfigMap + configMapKeyRef |
| 3 | Behavior khi apply PVC | AKS tự động provision qua StorageClass |
| 4 | Field để reference Secret trong Deployment | `valueFrom` với `secretKeyRef` |
| 5 | ConfigMap env var vs volume mount cho JSON config | Volume mount (file on disk) |

---

*Module hoàn thành. AKS Learning Path tiếp theo: Module 3*

---

[← Bài 3](./aks-m2-bai3-persistent-storage.md) · [🏠 Mục lục](../README.md) · [Bài 1 →](./aks-m3-bai1-monitor-logs-metrics.md)
