# Bài 3 — Attach Persistent Storage to an App

> Khoá: AI-200 · AKS — Configure applications on Azure Kubernetes Service

---

## Vấn đề: Container filesystem là ephemeral

Container filesystem bị xoá khi Pod restart hoặc di chuyển sang node khác. Với AI service, điều này ảnh hưởng đến:
- **Embeddings cache** — phải regenerate mỗi khi restart
- **Temporary artifacts** — mất file đang xử lý
- **Conversation state** — mất session data
- **User uploads** — mất file người dùng đã upload

**PersistentVolume (PV) và PersistentVolumeClaim (PVC)** cung cấp storage tồn tại qua Pod restart và rescheduling.

---

## Hai approach cho persistent storage trong AKS

### CSI Drivers (cách phổ biến)

Tích hợp Azure storage services qua **Container Storage Interface** — chuẩn Kubernetes. AKS có built-in StorageClasses cho:
- **Azure Disk** → block storage, single-node access
- **Azure Files** → SMB file share, multi-node shared access
- **Azure Blob** → large unstructured dataset

### Azure Container Storage (nâng cao)

Container-native storage platform tối ưu cho stateful workload, dùng NVMe-oF protocol. Giảm Pod failover time, tốt hơn cho I/O intensive app. Module này focus vào CSI Drivers.

---

## Built-in StorageClasses

| StorageClass | Backing storage | Access mode | Hiệu năng | Dùng khi |
|---|---|---|---|---|
| `managed-csi` (default) | Azure Disk | ReadWriteOnce | Standard HDD/SSD | Single Pod, database, cost-efficient |
| `managed-csi-premium` | Azure Disk Premium | ReadWriteOnce | Premium SSD | Single Pod, low latency, I/O intensive |
| `azurefile-csi` | Azure Files | **ReadWriteMany** | Standard | Nhiều Pod chia sẻ, logs, content |
| `azurefile-csi-premium` | Azure Files Premium | **ReadWriteMany** | Premium SSD | Nhiều Pod, high performance shared |

**Điểm quan trọng:**
- **Azure Disk** → `ReadWriteOnce` — disk attach vào một node, chỉ Pod trên node đó access được
- **Azure Files** → `ReadWriteMany` — SMB share mount trên nhiều node, nhiều Pod cùng đọc/ghi

---

## PersistentVolumeClaim (PVC) — Request storage

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
  - ReadWriteOnce          # Single node access
  resources:
    requests:
      storage: 10Gi        # Yêu cầu 10GB storage
  storageClassName: default  # Dùng default StorageClass (managed-csi)
```

Khi apply PVC → AKS dùng StorageClass để **tự động provision Azure storage**. Không cần tạo Azure Disk thủ công trong portal.

**Trạng thái PVC:**
- `Pending` → chờ được bind với PV
- `Bound` → đã được bind, sẵn sàng dùng
- `Released` → PVC bị xoá nhưng PV chưa reclaimed

---

## Mount PVC vào Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-api
  template:
    metadata:
      labels:
        app: web-api
    spec:
      volumes:
      - name: data-volume
        persistentVolumeClaim:
          claimName: data-pvc      # Tên PVC đã tạo
      containers:
      - name: api
        image: myregistry.azurecr.io/web-api:v1
        volumeMounts:
        - name: data-volume
          mountPath: /app/data     # Path trong container
```

App ghi file vào `/app/data` → file persist qua Pod restart.

> **Lưu ý với `replicas: 2` và ReadWriteOnce:** Azure Disk chỉ attach vào một node. Nếu 2 Pod trên cùng một node → OK. Nếu trên khác node → Pod thứ 2 sẽ Pending vì disk đang được dùng bởi node khác. **Dùng ReadWriteMany (Azure Files) khi cần nhiều Pod cùng access**.

---

## Verify Persistence

```bash
kubectl apply -f pvc.yaml
kubectl apply -f deployment.yaml

# Kiểm tra PVC đã Bound chưa
kubectl describe pvc data-pvc

# Kiểm tra Pod đang chạy
kubectl get pods -l app=web-api

# Test: ghi file vào Pod
kubectl exec <pod-name> -- sh -c "echo 'test data' > /app/data/test.txt"

# Delete Pod (Kubernetes tự tạo lại)
kubectl delete pod <pod-name>

# Verify file còn sau khi Pod mới start
kubectl exec <new-pod-name> -- cat /app/data/test.txt
# Nên thấy "test data"
```

---

## So sánh: ConfigMap vs Secret vs PVC

| Resource | Dùng cho | Sensitive? | Persist? |
|---|---|---|---|
| **ConfigMap** | Feature flags, endpoints, config files | ❌ Non-sensitive | ❌ (config, không phải data) |
| **Secret** | API key, password, TLS cert | ✅ Yes | ❌ (credential, không phải data) |
| **PVC** | Files, embeddings, artifacts, uploads | Neutral | ✅ Yes — survive Pod restart |

---

## Bản chất bài này là gì?

**Một câu:** PVC là cách khai báo "tôi cần X GB storage" trong Kubernetes — platform tự provision Azure Disk hoặc Azure Files phù hợp, giống Docker volume nhưng có access mode concept (ReadWriteOnce vs ReadWriteMany) quan trọng khi scale.

### So sánh với Docker

| Thao tác | Docker | AKS |
|---|---|---|
| Mount persistent storage | `docker run -v /host/path:/container/path` | PVC + volumeMounts trong Deployment |
| Tạo named volume | `docker volume create myvolume` | Apply PVC manifest (AKS tự provision Azure storage) |
| Shared volume nhiều container | `--volumes-from` | Azure Files PVC với `ReadWriteMany` |
| 1 container/1 volume | Mặc định | Azure Disk với `ReadWriteOnce` |
| Volume tồn tại sau container xoá | ✅ Named volume | ✅ PVC (tùy reclaim policy) |

**ReadWriteOnce vs ReadWriteMany — điểm khác biệt quan trọng nhất:**
- `ReadWriteOnce` (Azure Disk) = disk attach vào **một node** — nếu 2 Pod trên 2 node khác nhau, Pod thứ 2 sẽ `Pending`
- `ReadWriteMany` (Azure Files) = SMB share mount trên **nhiều node** — an toàn khi `replicas > 1`

> Nếu `replicas: 2` + Azure Disk → rủi ro cao: 2 Pod trên cùng node thì ok, nhưng rescheduling ra node khác → Pod bị Pending.

---

## Checklist ghi nhớ cho AI-200

- [ ] Container filesystem **ephemeral** — mất khi Pod restart
- [ ] **PVC** = request storage · **PV** = actual storage resource · **StorageClass** = provisioner
- [ ] AKS tự động provision Azure storage khi apply PVC — không cần tạo thủ công
- [ ] **Azure Disk** → `ReadWriteOnce` (một node) · **Azure Files** → `ReadWriteMany` (nhiều node)
- [ ] `managed-csi` (default) = Azure Disk Standard · `azurefile-csi` = Azure Files Standard
- [ ] Premium variants: `managed-csi-premium` và `azurefile-csi-premium`
- [ ] PVC states: `Pending` → `Bound` → sẵn sàng dùng
- [ ] `replicas > 1` + ReadWriteOnce + khác node → Pod thứ 2 Pending
- [ ] Verify persistence: ghi file → delete Pod → confirm file còn trong Pod mới

---

*Module Assessment tiếp theo*

---

[← Bài 2](./aks-m2-bai2-secrets.md) · [🏠 Mục lục](../README.md) · [Assessment →](./aks-m2-module-assessment.md)
