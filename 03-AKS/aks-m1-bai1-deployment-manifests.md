# Bài 1 — Create Kubernetes Deployment Manifests

> Khoá: AI-200 · AKS — Deploy applications to Azure Kubernetes Service

---

## AKS khác gì so với ACA?

Trước khi vào chi tiết, cần hiểu tại sao chọn AKS thay vì ACA:

| | ACA | AKS |
|---|---|---|
| Kiểm soát infrastructure | Thấp — managed hoàn toàn | Cao — bạn quản lý cluster |
| Kubernetes trực tiếp | Ẩn sau abstraction | ✅ Trực tiếp dùng kubectl + YAML |
| Độ phức tạp | Thấp | Cao hơn |
| Phù hợp | Microservice, serverless | Workload phức tạp, control plane cần tùy chỉnh |

AKS là managed Kubernetes trên Azure — Azure quản lý control plane, bạn quản lý node và application.

---

## 4 Khái niệm cốt lõi

Trước khi viết manifest, cần hiểu 4 building block:

**Pod** — Đơn vị nhỏ nhất trong Kubernetes. Một Pod bọc container của bạn. Không quản lý Pod trực tiếp — để Deployment làm việc đó.

**Deployment** — Khai báo "tôi muốn N bản copy của app này chạy". Deployment monitor Pod health và tự restart Pod crash. Bạn viết Deployment manifest để mô tả desired state.

**Service** — Cung cấp stable endpoint để access Pod. Pod IP thay đổi mỗi khi restart — Service tạo ra địa chỉ cố định, tự route traffic đến đúng Pod.

**kubectl** — CLI tool tương tác với Kubernetes cluster. Dùng để apply manifest, check status, xem log, troubleshoot.

---

## Cấu trúc Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-inference-api       # Tên Deployment
  namespace: default           # Namespace (default cho hầu hết trường hợp)
spec:
  replicas: 2                  # Số Pod cần chạy
  selector:
    matchLabels:
      app: inference-api       # Deployment quản lý Pod có label này
  template:
    metadata:
      labels:
        app: inference-api     # Pod nhận label này — phải match selector ở trên
    spec:
      containers:
      - name: api
        image: myregistry.azurecr.io/inference-api:v1.0
        ports:
        - containerPort: 8080  # Port app lắng nghe bên trong container
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"       # 1000 millicores = 1 CPU core
          limits:
            memory: "4Gi"
            cpu: "2000m"
        env:
        - name: MODEL_NAME
          value: "gpt-4"
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: api-secrets    # Tên Secret object
              key: api-key         # Key trong Secret
```

---

## Container Image

Format đường dẫn image: `registryname.azurecr.io/imagename:tag`

```yaml
containers:
- name: api
  image: myregistry.azurecr.io/inference-api:v1.0
```

Đảm bảo image đã được:
- Build và push lên registry
- Accessible từ AKS cluster (AKS + ACR trong cùng subscription thường tự động có quyền)
- Bao gồm toàn bộ application code và dependencies

---

## Replicas — Tại sao cần nhiều hơn 1?

```yaml
spec:
  replicas: 2
```

Nếu chỉ có 1 Pod và nó crash → app down cho đến khi Kubernetes detect và restart. Với 2-3 replicas, Pod còn lại vẫn xử lý request trong khi Pod crash được restart.

**Guideline cho AI inference API:**

| Replicas | Phù hợp |
|---|---|
| 2 | Basic resilience, dev và non-critical service |
| 3 | Production API phục vụ external user — recommended |
| 4+ | High traffic cần load distribution — cân nhắc cost |

Mỗi replica tốn resource — balance giữa resilience và cost.

---

## Resource Requests và Limits

Kubernetes cần biết app cần bao nhiêu resource:

```yaml
resources:
  requests:
    memory: "2Gi"    # Cần 2GB để chạy — dùng để schedule Pod lên node có đủ resource
    cpu: "1000m"     # Cần 1 CPU core (1000 millicores)
  limits:
    memory: "4Gi"    # Không được vượt 4GB — bị terminate nếu vượt (OOMKill)
    cpu: "2000m"     # Không được vượt 2 CPU core — bị throttle nếu vượt
```

**Requests** — Lượng resource cần để start. Kubernetes dùng giá trị này để schedule Pod. Nếu node không có đủ → Pod ở trạng thái Pending.

**Limits** — Mức tối đa Pod được dùng. Memory vượt limit → bị **terminate và restart (OOMKill)**. CPU vượt limit → bị **throttle** (không terminate).

**Guideline cho AI inference:**
- Memory requests: Dựa trên model size + thêm 20% overhead. Model nhỏ: 2-4Gi. Model lớn: nhiều hơn.
- CPU requests: Thường 1-2 CPU core/Pod cho inference workload.
- Requests quá thấp → OOMKill. Requests quá cao → lãng phí resource và tiền.

**Đơn vị CPU:** `1000m` = 1 core. `500m` = 0.5 core. `250m` = 0.25 core.

---

## Environment Variables và Secrets

### Non-sensitive config — dùng env vars

```yaml
env:
- name: MODEL_NAME
  value: "gpt-4"
- name: API_ENDPOINT
  value: "https://api.example.com"
```

### Sensitive data — dùng Kubernetes Secrets

```yaml
env:
- name: API_KEY
  valueFrom:
    secretKeyRef:
      name: api-secrets      # Tên Secret object (tạo riêng bằng kubectl)
      key: api-key           # Key trong Secret
```

Tạo Secret bằng kubectl:

```bash
kubectl create secret generic api-secrets --from-literal=api-key=your-secret-key
```

> **Không bao giờ hardcode secret trực tiếp trong manifest.** Secret trong manifest → commit lên Git → lộ credential vĩnh viễn trong history.

---

## Label và Selector — Kết nối Deployment với Service

Label là key-value metadata gắn vào Pod. Selector là cách Deployment (và Service) tìm Pod để quản lý.

```yaml
# Trong Deployment — Pods nhận label này
template:
  metadata:
    labels:
      app: inference-api     # ← Pod có label này

# Trong Deployment spec — Deployment quản lý Pod có label này
selector:
  matchLabels:
    app: inference-api       # ← phải khớp với label trên
```

Label phải khớp với selector. Nếu không khớp → Deployment không biết Pod nào để quản lý.

---

## Bản chất bài này là gì?

**Một câu:** Kubernetes manifest là cách "nói chuyện" với Kubernetes bằng YAML — bạn mô tả **desired state** (muốn 2 replica chạy), Kubernetes liên tục làm cho thực tế khớp với desired state đó.

### So sánh với Docker

| Khái niệm | Docker | Kubernetes/AKS |
|---|---|---|
| Chạy container | `docker run image:tag` | Deployment manifest + `kubectl apply` |
| Nhiều instance | Docker Compose `scale` | `replicas: N` trong Deployment |
| Restart khi crash | `--restart=always` | Deployment tự động (always) |
| Biến môi trường | `--env KEY=VALUE` | `env:` trong container spec |
| Secret | `docker secret create` / `-e` | `secretKeyRef` từ Secret object |
| CPU/Memory limit | `--cpus 1.0 --memory 2g` | `resources.limits` (CPU + memory tách biệt) |

**Điểm khác biệt cốt lõi — Docker là imperative, Kubernetes là declarative:**
- Docker: `docker run`, `docker stop`, `docker restart` — bạn ra lệnh từng bước
- Kubernetes: bạn khai báo "tôi muốn 2 Pod", platform tự làm và duy trì

**requests vs limits — không có trong Docker thông thường:** Docker chỉ có limits (`--memory`, `--cpus`). Kubernetes tách ra:
- `requests` = Kubernetes dùng để **schedule** Pod lên node có đủ resource
- `limits` = mức tối đa Pod được dùng (memory vượt → OOMKill, CPU vượt → throttle)

---

## Checklist ghi nhớ cho AI-200

- [ ] **Pod** = smallest unit · **Deployment** = manages Pods · **Service** = stable endpoint · **kubectl** = CLI tool
- [ ] Deployment manifest: `apiVersion`, `kind`, `metadata`, `spec` (replicas, selector, template)
- [ ] Image format: `registryname.azurecr.io/imagename:tag`
- [ ] **`replicas: 2-3`** cho production AI API
- [ ] **Requests** = lượng resource cần để schedule · **Limits** = maximum
- [ ] Memory vượt limit → **OOMKill** (terminate) · CPU vượt limit → **throttle** (slow)
- [ ] CPU: `1000m` = 1 core · `500m` = 0.5 core
- [ ] Env vars cho non-sensitive · `secretKeyRef` cho sensitive data
- [ ] **Không hardcode secret trong manifest** — dùng `kubectl create secret`
- [ ] Pod labels trong template phải **match** với `selector.matchLabels`

---

[← Mục lục](../README.md) · [🏠 Mục lục](../README.md) · [Bài 2 →](./aks-m1-bai2-expose-applications.md)
