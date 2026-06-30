# Bài 3 — Deploy Applications to Azure Kubernetes Service

> Khoá: AI-200 · AKS — Deploy applications to Azure Kubernetes Service

---

## Deploy bằng kubectl apply

`kubectl apply` là lệnh chính để deploy lên AKS. Nó đọc YAML manifest và tạo hoặc update resource:

```bash
# Deploy từng file
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Deploy nhiều file cùng lúc
kubectl apply -f deployment.yaml -f service.yaml

# Deploy tất cả YAML trong thư mục hiện tại
kubectl apply -f .
```

**`apply` là idempotent** — chạy lại cùng file sẽ update resource nếu có thay đổi, không làm gì nếu đã đúng. Khác với `kubectl create` chỉ tạo mới và báo lỗi nếu đã tồn tại.

**Deployment là asynchronous:** `kubectl apply` return ngay lập tức, nhưng Pod cần vài giây để start (pull image, init container). Phải verify sau đó.

---

## Verify Deployment Status

### Kiểm tra Pod

```bash
kubectl get pods

# Output:
# NAME                             READY  STATUS    RESTARTS   AGE
# inference-api-7d8b4f9c6-abc12    1/1    Running   0          2m
# inference-api-7d8b4f9c6-xyz78    1/1    Running   0          2m
```

**READY:** `1/1` = container trong Pod đang chạy. `0/1` = container chưa ready.

**STATUS:**
- `Running` ✅ — Bình thường
- `Pending` ⏳ — Đang chờ: pull image hoặc chờ resource
- `CrashLoopBackOff` ❌ — App crash liên tục — cần xem log
- `ImagePullBackOff` ❌ — Không pull được image

**RESTARTS:** Số lần Pod đã restart. Số cao = app đang có vấn đề.

### Kiểm tra Deployment

```bash
kubectl get deployment

# Output:
# NAME             READY   UP-TO-DATE   AVAILABLE   AGE
# inference-api    2/2     2            2           2m
```

`2/2` trong READY = cả 2 replicas đang chạy. Nếu `1/2` = có 1 Pod chưa ready.

### Kiểm tra Service

```bash
kubectl get svc

# Output:
# NAME                      TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)
# inference-api-service     LoadBalancer   10.0.12.45    40.89.123.45   80:30523/TCP
```

**EXTERNAL-IP** (LoadBalancer) = public IP client dùng. `<pending>` = Azure đang provision.

---

## Xem Logs

```bash
# Log của Pod cụ thể
kubectl logs inference-api-7d8b4f9c6-abc12

# Log 10 phút gần nhất
kubectl logs inference-api-7d8b4f9c6-abc12 --since=10m

# Log của Pod đã crash (lần chạy trước)
kubectl logs inference-api-7d8b4f9c6-abc12 --previous

# Log từ tất cả Pod trong Deployment (dùng label selector)
kubectl logs -l app=inference-api
```

`--previous` quan trọng khi Pod đang CrashLoopBackOff — log hiện tại chỉ có từ lần restart mới nhất, `--previous` cho thấy lỗi từ lần crash trước.

---

## Test Connectivity

### ClusterIP (internal) — dùng debug Pod

```bash
# Tạo debug Pod tạm thời
kubectl run -it --rm debug --image=alpine:latest --restart=Never -- sh

# Từ bên trong debug Pod
wget http://inference-api-service:80
```

### LoadBalancer (external) — dùng curl từ máy local

```bash
# Lấy public IP
kubectl get svc inference-api-service

# Test với curl
curl http://40.89.123.45

# Test endpoint cụ thể
curl http://40.89.123.45/predict -X POST -d '{"input": "test"}'
```

---

## Troubleshoot Common Issues

### ImagePullBackOff

**Triệu chứng:** Pod status = `ImagePullBackOff`

**Nguyên nhân:** Kubernetes không pull được image từ registry.

**Diagnose:**
```bash
kubectl describe pod inference-api-7d8b4f9c6-abc12
# Xem phần Events — thấy lỗi như "Failed to pull image: image not found"
```

**Giải pháp:**
- Verify image path và tag đúng không
- Confirm image đã push lên registry: `az acr repository list --name myregistry`
- Đảm bảo AKS có quyền pull từ ACR (thường tự động nếu cùng subscription)
- Test pull manual: `docker pull myregistry.azurecr.io/inference-api:v1.0`

---

### CrashLoopBackOff

**Triệu chứng:** Pod status = `CrashLoopBackOff` — app crash liên tục.

**Nguyên nhân:** App exit hoặc crash khi startup.

**Diagnose:**
```bash
# Xem log lần crash trước
kubectl logs inference-api-7d8b4f9c6-abc12 --previous

# Xem events
kubectl describe pod inference-api-7d8b4f9c6-abc12
```

**Giải pháp:**
- Đọc log để tìm error message (missing env var, config error...)
- Verify tất cả env var và secret đã được configure đúng
- Test image local: `docker run myregistry.azurecr.io/inference-api:v1.0`

---

### Pending Pods

**Triệu chứng:** Pod stuck ở `Pending`, không chuyển sang Running.

**Nguyên nhân:** Kubernetes không schedule được Pod — không đủ resource (CPU/memory) trên node.

**Diagnose:**
```bash
kubectl describe pod inference-api-7d8b4f9c6-abc12
# Xem Events: "Insufficient memory" hoặc "Insufficient cpu"

kubectl get nodes
kubectl describe node <node-name>
```

**Giải pháp:**
- Giảm resource requests trong manifest nếu đang request quá nhiều
- Scale up cluster: `az aks scale --resource-group mygroup --name myaks --node-count 3`
- Check `kubectl top nodes` để xem actual resource usage (cần metrics server)

---

### Service Has No Endpoints

**Triệu chứng:** Service tồn tại nhưng `kubectl describe svc` hiện `Endpoints: <none>` → traffic không đến đâu.

**Nguyên nhân:** Không có Pod nào match selector của Service.

**Diagnose:**
```bash
# Xem Pod labels
kubectl get pods --show-labels

# Xem Service selector và endpoints
kubectl describe svc inference-api-service
# So sánh "Selector" với labels của Pod
```

**Giải pháp:**
- Đảm bảo Pod label trong Deployment template khớp với Service selector
- Đây là lỗi typo phổ biến: `app: inference-api` vs `app: inferenceapi`

---

## Tóm tắt luồng deploy hoàn chỉnh

```bash
# 1. Apply manifests
kubectl apply -f deployment.yaml -f service.yaml

# 2. Check Pod status (chờ Running)
kubectl get pods

# 3. Check Deployment (2/2 READY)
kubectl get deployment

# 4. Check Service (lấy EXTERNAL-IP)
kubectl get svc

# 5. Test connectivity
curl http://<EXTERNAL-IP>

# Nếu có vấn đề → xem log
kubectl logs <pod-name>
kubectl describe pod <pod-name>
```

---

## Bản chất bài này là gì?

**Một câu:** `kubectl apply` + verify loop là workflow cốt lõi của AKS — bạn khai báo muốn gì, sau đó verify Kubernetes có đạt được trạng thái đó không, và dùng log/describe để debug khi không đạt được.

### So sánh với Docker

| Thao tác | Docker | kubectl |
|---|---|---|
| Chạy app | `docker run image` | `kubectl apply -f deployment.yaml` |
| Xem trạng thái | `docker ps` | `kubectl get pods` / `kubectl get deployment` |
| Xem log | `docker logs container` | `kubectl logs pod-name` |
| Log real-time | `docker logs -f container` | `kubectl logs -f pod-name` |
| **Log lần crash trước** | ❌ Không có | `kubectl logs --previous` ✅ |
| Debug chi tiết | `docker inspect container` | `kubectl describe pod pod-name` |
| Lỗi phổ biến | Container not found | ImagePullBackOff, CrashLoopBackOff, Pending |

**`kubectl logs --previous` là điểm khác biệt quan trọng:** Docker không lưu log của container đã bị replace. Kubernetes giữ log của container instance trước trong cùng Pod — khi Pod đang CrashLoopBackOff, đây là cách duy nhất thấy được error gây ra crash.

**4 trạng thái Pod cần thuộc:** `Running` (ok), `Pending` (chờ resource), `CrashLoopBackOff` (app crash), `ImagePullBackOff` (không kéo được image).

---

## Checklist ghi nhớ cho AI-200

- [ ] `kubectl apply -f` — create hoặc update resource từ manifest (idempotent)
- [ ] `kubectl apply -f .` — apply tất cả YAML trong thư mục
- [ ] Pod STATUS: `Running` ✅ · `Pending` ⏳ · `CrashLoopBackOff` ❌ · `ImagePullBackOff` ❌
- [ ] `kubectl get pods` · `kubectl get deployment` · `kubectl get svc`
- [ ] `kubectl logs <pod>` · `kubectl logs <pod> --previous` (crash log) · `kubectl logs -l <label>`
- [ ] `kubectl describe pod <pod>` → xem Events để diagnose
- [ ] **ImagePullBackOff** → image path sai hoặc không có quyền pull
- [ ] **CrashLoopBackOff** → app crash, xem log `--previous`
- [ ] **Pending** → không đủ resource trên node
- [ ] **No Endpoints** → selector không match Pod labels
- [ ] LoadBalancer EXTERNAL-IP `<pending>` → chờ Azure provision

---

*Module Assessment tiếp theo*

---

[← Bài 2](./aks-m1-bai2-expose-applications.md) · [🏠 Mục lục](../README.md) · [Assessment →](./aks-m1-module-assessment.md)
