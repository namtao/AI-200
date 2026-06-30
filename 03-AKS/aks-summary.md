# Tổng hợp kiến thức — 03-AKS (3 Modules)

> AI-200 · Module 1: Deploy applications to AKS · Module 2: Configure applications on AKS · Module 3: Monitor and troubleshoot AKS

---

## Bức tranh tổng thể

```
AKS vs ACA — khi nào dùng cái nào?

ACA: serverless container, không cần quản lý cluster, tốt cho microservice + event-driven
AKS: managed Kubernetes, cần control plane tùy chỉnh, GPU, compliance, workload phức tạp

AKS = Azure quản lý control plane, bạn quản lý node và application.
Bạn dùng kubectl + YAML manifest trực tiếp — không có abstraction layer như ACA.
```

**Kubernetes là declarative, Docker là imperative:**
- Docker: `docker run`, `docker stop` → ra lệnh từng bước
- Kubernetes: khai báo desired state → platform duy trì liên tục

**3 module = 3 giai đoạn:**
```
Module 1: Deploy      → Manifest → Service → kubectl apply + verify
Module 2: Configure   → ConfigMap → Secrets → PVC
Module 3: Monitor     → Logs + Metrics → Troubleshoot → Connectivity
```

---

## MODULE 1 — Deploy Applications to AKS

### 4 Khái niệm cốt lõi

| Khái niệm | Vai trò | Tương đương Docker |
|---|---|---|
| **Pod** | Đơn vị nhỏ nhất, bọc container | `docker run` → 1 container |
| **Deployment** | Quản lý số lượng Pod, tự restart khi crash | `docker run --restart=always` + orchestration |
| **Service** | Stable endpoint cho Pod fleet | `docker run -p` + load balancer |
| **kubectl** | CLI tương tác với cluster | `docker` CLI |

### Deployment Manifest — Cấu trúc

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-inference-api
spec:
  replicas: 2                        # Số Pod muốn chạy
  selector:
    matchLabels:
      app: inference-api             # Deployment quản lý Pod có label này
  template:
    metadata:
      labels:
        app: inference-api           # Pod nhận label này — PHẢI match selector
    spec:
      containers:
      - name: api
        image: myregistry.azurecr.io/inference-api:v1.0
        ports:
        - containerPort: 8080
        resources:
          requests:                  # Dùng để SCHEDULE Pod lên node có đủ resource
            memory: "2Gi"
            cpu: "1000m"             # 1000m = 1 core
          limits:                    # Maximum — memory vượt → OOMKill, CPU vượt → throttle
            memory: "4Gi"
            cpu: "2000m"
        env:
        - name: MODEL_NAME
          value: "gpt-4"
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: api-key
```

**Label phải khớp selector:** `template.metadata.labels` = `selector.matchLabels`. Nếu không khớp → Deployment không quản lý được Pod nào.

### Service — 4 Types

| Type | Accessible từ | Dùng khi |
|---|---|---|
| **ClusterIP** (default) | Trong cluster only | Internal service: vector DB, embedding API, cache |
| **NodePort** | Node IP + port 30000-32767 | Dev/testing |
| **LoadBalancer** | Internet + Azure LB tự động | Production external AI API |
| **ExternalName** | Map DNS | Ít dùng |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: inference-api-service
spec:
  type: LoadBalancer
  selector:
    app: inference-api          # Route đến Pod có label này — PHẢI match Deployment labels
  ports:
  - port: 80                    # Client kết nối port này
    targetPort: 8080            # App lắng nghe port này trong container
```

**Port vs targetPort:** `port` = external (client dùng) · `targetPort` = container internal.  
**DNS nội bộ:** `http://service-name.namespace.svc.cluster.local:port` (hoặc ngắn hơn: `http://service-name:port` trong cùng namespace).

### kubectl apply — Deploy Workflow

```bash
# Apply manifest
kubectl apply -f deployment.yaml -f service.yaml
kubectl apply -f .                        # Tất cả YAML trong thư mục

# Verify
kubectl get pods                          # STATUS + READY + RESTARTS
kubectl get deployment                    # READY 2/2
kubectl get svc                           # EXTERNAL-IP của LoadBalancer

# Debug
kubectl logs <pod-name>
kubectl logs <pod-name> --previous        # Log lần crash trước (CrashLoopBackOff)
kubectl logs -l app=inference-api         # Log tất cả Pod cùng label
kubectl describe pod <pod-name>           # Events + config + state
```

**4 Pod states:** `Running` ✅ · `Pending` ⏳ (resource không đủ) · `CrashLoopBackOff` ❌ (app crash) · `ImagePullBackOff` ❌ (không pull image)

---

## MODULE 2 — Configure Applications on AKS

### ConfigMap vs Secret vs PVC

| Resource | Dùng cho | Sensitive? | Persist data? |
|---|---|---|---|
| **ConfigMap** | Feature flags, endpoints, config files | ❌ | ❌ |
| **Secret** | API key, password, TLS cert, registry cred | ✅ | ❌ |
| **PVC** | Files, embeddings, uploads, artifacts | Neutral | ✅ |

### ConfigMap — Non-sensitive Config

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-settings
data:
  FEATURE_X_ENABLED: "true"
  SERVICE_ENDPOINT: "https://api.example.com"
  config.json: |
    {"log_level": "info", "timeout": 30}
```

**Inject vào Pod — 3 cách:**

```yaml
# Cách 1: Từng key → env var
env:
- name: FEATURE_X_ENABLED
  valueFrom:
    configMapKeyRef:
      name: app-settings
      key: FEATURE_X_ENABLED

# Cách 2: Tất cả keys → env vars (envFrom)
envFrom:
- configMapRef:
    name: app-settings

# Cách 3: Mount như files
volumes:
- name: config-vol
  configMap:
    name: app-settings
containers:
- volumeMounts:
  - name: config-vol
    mountPath: /app/config
```

| | Env var | Volume mount |
|---|---|---|
| Auto-update khi ConfigMap thay đổi | ❌ cần restart Pod | ✅ tự sync (trừ subPath) |
| App đọc từ | `os.environ` | file trên disk |

**Giới hạn 1 MiB** per ConfigMap. `immutable: true` → không sửa được, phải delete + recreate.

### Secret — Sensitive Data

```bash
# Tạo secret
kubectl create secret generic app-secrets \
  --from-literal=API_KEY="your-key" \
  --from-literal=DB_CONNECTION="Host=db;Password=secure"
```

```yaml
# Reference trong Deployment
env:
- name: API_KEY
  valueFrom:
    secretKeyRef:
      name: app-secrets
      key: API_KEY
```

**3 Secret types:** `generic` (opaque) · `kubernetes.io/dockerconfigjson` (registry) · `kubernetes.io/tls`  
**Base64 không phải encryption** — chỉ là encoding. Không commit YAML có secret value lên Git.

**Production — Azure Key Vault:**

| Option | Secret ở đâu | Audit | Auto-rotate |
|---|---|---|---|
| CSI Driver + Key Vault | Key Vault (không đi qua etcd) | ✅ Fine-grained | ✅ |
| App Config + Key Vault | Key Vault → K8s Secret (đi qua etcd) | ⚠️ Less granular | ✅ |

### Persistent Storage — PVC

Container filesystem = **ephemeral** (mất khi Pod restart). PVC = persistent storage request.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
  - ReadWriteOnce          # Azure Disk: single node
  resources:
    requests:
      storage: 10Gi
  storageClassName: managed-csi   # AKS tự provision Azure Disk
```

| StorageClass | Backing | Access Mode | Dùng khi |
|---|---|---|---|
| `managed-csi` (default) | Azure Disk | ReadWriteOnce | Single Pod, database |
| `managed-csi-premium` | Azure Disk Premium | ReadWriteOnce | Low latency, I/O intensive |
| `azurefile-csi` | Azure Files | **ReadWriteMany** | Nhiều Pod chia sẻ |
| `azurefile-csi-premium` | Azure Files Premium | **ReadWriteMany** | Nhiều Pod, high perf |

**`replicas > 1` + ReadWriteOnce + khác node → Pod thứ 2 Pending.** Dùng `ReadWriteMany` (Azure Files) khi cần nhiều Pod.

```yaml
# Mount PVC vào Deployment
volumes:
- name: data-volume
  persistentVolumeClaim:
    claimName: data-pvc
containers:
- volumeMounts:
  - name: data-volume
    mountPath: /app/data
```

---

## MODULE 3 — Monitor and Troubleshoot

### Monitoring — Portal vs kubectl

| | Azure Portal | kubectl |
|---|---|---|
| Dùng khi | Visual overview, chia sẻ team | Targeted query, scripting |
| Logs | Live Logs streaming | `kubectl logs` |
| Metrics | Container insights dashboard | `kubectl top` |
| Filter | Namespace, Pod, container | `-n namespace`, `-l label` |

```bash
# Logs
kubectl logs <pod-name> -n ai-workloads
kubectl logs -f <pod-name>                # Follow real-time
kubectl logs -l app=inference-api         # Tất cả Pod cùng label
kubectl logs <pod-name> -c <container>    # Pod nhiều container

# Metrics (cần metrics server)
kubectl top nodes
kubectl top pods -n ai-workloads
```

**4 key signals cho AI workload:** Response latency · Error rate (5xx/timeout) · Pod restart count · CPU/Memory vs limits

### Troubleshoot Workflow (6 bước)

```
1. kubectl get pods -n ns
   → Identify state: Running/CrashLoopBackOff/Pending/ImagePullBackOff
        ↓
2. kubectl describe pod <name> -n ns
   → Events: image pull fail? probe fail? scheduling issue? OOMKill?
   → Exit code 137 = OOMKill | khác = app error
        ↓
3. kubectl logs <name> --previous
   → Error message từ lần crash trước
        ↓
4. kubectl exec -it <name> -- /bin/sh
   → Verify: env vars, files, connectivity nội bộ
        ↓
5. kubectl describe service <name> -n ns
   → Selector match? Port/targetPort đúng? Endpoints có không?
        ↓
6. kubectl get endpointslices -l kubernetes.io/service-name=<name>
   → Pod IPs thực sự nhận traffic (ground truth)
```

**Quy tắc vàng:** Diagnose bên trong container (`exec`), fix bằng manifest (`kubectl apply`). Thay đổi trong container biến mất khi Pod restart.

### Connectivity Verification

```bash
# Port-forward — test local mà không expose external
kubectl port-forward service/inference-api 8080:80 -n ai-workloads
# Terminal khác:
curl http://localhost:8080/api/inference

# External IP
kubectl get svc inference-api -n ai-workloads   # EXTERNAL-IP
kubectl get ingress -n ai-workloads             # Address
```

**Port-forward use cases:** Debug model endpoint trước khi expose · Ingress chưa setup · Không muốn expose production.

---

## CLI Commands Quick Reference

```bash
# === Cluster access ===
az aks get-credentials --resource-group rg --name my-aks-cluster
kubectl config get-contexts

# === Namespace ===
kubectl get namespaces
kubectl get pods -n ai-workloads          # Xem namespace cụ thể
kubectl get pods -A                       # Tất cả namespace

# === Deploy ===
kubectl apply -f deployment.yaml -f service.yaml
kubectl apply -f .                        # Tất cả YAML trong thư mục

# === Verify ===
kubectl get pods
kubectl get deployment
kubectl get svc
kubectl get pods --show-labels
kubectl get pods -o wide                  # Thêm NODE và IP

# === Debug ===
kubectl describe pod <name>
kubectl describe svc <name>
kubectl get events -n <namespace>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl logs -f <pod-name>
kubectl logs -l app=<label>
kubectl exec -it <pod-name> -- /bin/sh

# === Metrics ===
kubectl top nodes
kubectl top pods

# === Config ===
kubectl apply -f configmap.yaml
kubectl describe configmap <name>
kubectl exec <pod> -- printenv | grep KEY

# === Secret ===
kubectl create secret generic <name> --from-literal=KEY=VALUE
kubectl get secrets
kubectl describe secret <name>            # Keys + size, không expose value

# === Storage ===
kubectl apply -f pvc.yaml
kubectl describe pvc <name>               # Status: Pending/Bound

# === Connectivity ===
kubectl get endpointslices -l kubernetes.io/service-name=<name> -n <ns>
kubectl port-forward service/<name> 8080:80 -n <ns>
kubectl get ingress -n <namespace>

# === Scale ===
kubectl scale deployment <name> --replicas=3
az aks scale --resource-group rg --name my-aks --node-count 3
```

---

## Exam Traps — Những điểm hay nhầm

1. **Pod labels trong Deployment template phải match Service selector** — nếu không khớp, Service có `Endpoints: <none>` và traffic không đến Pod nào. Không có error message rõ ràng.

2. **`kubectl logs --previous` cho CrashLoopBackOff** — không có flag → chỉ thấy log lần chạy hiện tại (rất ngắn). `--previous` mới thấy error gây crash.

3. **Kubernetes không tự thêm node khi Pod Pending** — Cluster Autoscaler là add-on riêng. Mặc định Pod cứ Pending mãi cho đến khi có đủ resource.

4. **Base64 không phải encryption** — `kubectl get secret -o yaml` vẫn đọc được value. Dùng RBAC chặt và Key Vault cho production.

5. **`replicas > 1` + ReadWriteOnce (Azure Disk) + khác node → Pod Pending** — Disk chỉ mount trên 1 node. Dùng Azure Files (`ReadWriteMany`) khi multi-replica cần shared storage.

6. **ConfigMap env var không auto-update khi ConfigMap thay đổi** — phải restart Pod. Volume mount tự sync (trừ subPath).

7. **`containerPort` trong Deployment và `targetPort` trong Service phải khớp nhau** — nhưng Kubernetes không báo lỗi nếu không khớp, chỉ connection refused.

8. **`kubectl apply` là idempotent, `kubectl create` thì không** — `create` báo lỗi nếu resource đã tồn tại. Luôn dùng `apply` cho repeatability.

9. **Port-forward dừng khi terminal đóng** — chỉ là debug tool, không phải production traffic path.

10. **`kubectl describe pod` ≠ `kubectl logs`:** `describe` cho infrastructure events (scheduling, image pull, probe) · `logs` cho application output. CrashLoopBackOff → cần cả hai.

11. **Exit code 137 = OOMKill (memory vượt limit)** — tăng memory limit, không tăng CPU.

12. **Service với `Deployment name match` là sai** — Service không liên kết với Deployment theo tên, mà liên kết với Pod theo label selector.

---

## AKS vs ACA vs App Service — So sánh cuối

| | App Service | ACA | AKS |
|---|---|---|---|
| Kubernetes trực tiếp | ❌ | Ẩn | ✅ kubectl + manifest |
| Shell access (debug) | ✅ Kudu + SSH | ❌ | ✅ kubectl exec |
| GPU support | ❌ | ✅ Dedicated | ✅ |
| Scale-to-zero | ❌ | ✅ | Cần HPA + KEDA riêng |
| Persistent volume | ❌ | ❌ | ✅ PVC + Azure Disk/Files |
| Config management | App Settings / KV ref | Secrets + env vars | ConfigMap + Secret + PVC |
| Event-driven scaling | ❌ | ✅ KEDA native | KEDA add-on |
| Ingress control | ❌ | Partial | ✅ Full Ingress controller |
| Operational complexity | Thấp | Trung bình | Cao |
| Use case | Simple web app/API | Microservices, serverless | Complex workload, full control |

---

## Assessment Summary

### Module 1 — Deploy
| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Service type cho internet-facing app | LoadBalancer |
| 2 | Behavior khi resource requests vượt capacity | Pod stays Pending (không tự thêm node) |
| 3 | Điều kiện traffic routing | Pod labels match Service selector |
| 4 | Log của Pod đã crash | `kubectl logs --previous` |
| 5 | Ý nghĩa `replicas` field | Số Pod copies chạy đồng thời |

### Module 2 — Configure
| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Resource cho connection string có password | Secret |
| 2 | Inject non-sensitive config không rebuild image | ConfigMap + configMapKeyRef |
| 3 | Behavior khi apply PVC | AKS tự động provision qua StorageClass |
| 4 | Field để reference Secret trong Deployment | `valueFrom` + `secretKeyRef` |
| 5 | ConfigMap env var vs volume mount cho JSON file | Volume mount (app đọc từ file) |

### Module 3 — Monitor
| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Inspect log real-time khi reproduce HTTP 500 | `kubectl logs -f` |
| 2 | First step khi CrashLoopBackOff | `kubectl describe pod` + inspect events |
| 3 | Confirm Service selector match Pod labels | `kubectl describe service` |
| 4 | Test AI endpoint từ local trước khi expose | `kubectl port-forward service/...` |
| 5 | Fix CPU throttle gây latency tăng | Tăng CPU limit hoặc scale out replicas |

---

[← Assessment](./aks-m3-module-assessment.md) · [🏠 Mục lục](../README.md) · [Mục lục →](../README.md)
