# Module Assessment — Deploy Applications to Azure Kubernetes Service

> Khoá: AI-200 · AKS — Deploy applications to Azure Kubernetes Service

---

## Câu 1

**Which Service type should you use when you need to expose an application to the internet with an Azure-managed load balancer?**

- LoadBalancer ✅
- ClusterIP
- NodePort

**Đáp án: LoadBalancer**

`type: LoadBalancer` là Service type duy nhất trong danh sách tạo ra **Azure-managed load balancer với public IP**. Azure tự động provision Load Balancer, assign public IP, và configure health probe.

```yaml
spec:
  type: LoadBalancer    # Azure provision Load Balancer + public IP
  selector:
    app: inference-api
  ports:
  - port: 80
    targetPort: 8080
```

Tại sao các đáp án còn lại sai:
- **ClusterIP:** Internal only — chỉ accessible trong cluster, không có external IP. Dùng cho service nội bộ như vector DB hay embedding service.
- **NodePort:** Expose qua node IP và port cao (30000+) — không có managed load balancer. Thường dùng cho testing/dev, không phải production internet-facing app.

---

## Câu 2

**What happens if a Pod's resource requests exceed the available capacity on all cluster nodes?**

- The Pod stays in Pending status until sufficient resources become available ✅
- Kubernetes automatically scales the cluster to add more nodes
- The Pod starts but with reduced resource allocation

**Đáp án: Pod stays in Pending status**

Kubernetes **không tự động add node** (trừ khi Cluster Autoscaler được cấu hình riêng). Khi Pod requests nhiều resource hơn bất kỳ node nào available, Pod sẽ ở trạng thái **Pending** — Kubernetes liên tục check và sẽ schedule Pod khi có đủ resource.

```bash
kubectl describe pod <pod-name>
# Events sẽ hiện: "0/3 nodes are available: Insufficient memory"
```

Tại sao các đáp án còn lại sai:
- **Auto-scale cluster:** Đây là behavior của Cluster Autoscaler (add-on riêng) chứ không phải default. Kubernetes mặc định không tự thêm node.
- **Start with reduced allocation:** Kubernetes không thỏa hiệp resource — Pod yêu cầu 2Gi memory thì phải có đủ 2Gi mới schedule. Không "start với ít hơn".

---

## Câu 3

**What must match between a Deployment and a Service for traffic to route correctly?**

- The Pod labels in the Deployment template must match the Service selector ✅
- The container port in the Deployment must match the Service port
- The Deployment name must match the Service name

**Đáp án: Pod labels phải match Service selector**

Đây là cơ chế kết nối cốt lõi của Kubernetes. Service dùng **selector** để tìm Pod để route traffic đến:

```yaml
# Deployment template — Pod nhận label này
template:
  metadata:
    labels:
      app: inference-api     # ← Pod label

# Service — tìm Pod có label này
selector:
  app: inference-api         # ← phải khớp với Pod label
```

Nếu không khớp → Service có `Endpoints: <none>` → traffic không đến Pod nào.

Tại sao các đáp án còn lại sai:
- **Container port phải match Service port:** `containerPort` (trong Deployment) và `targetPort` (trong Service) nên match — nhưng đây không phải điều kiện bắt buộc của Kubernetes, chỉ là best practice. Service routing dựa vào selector, không phải port matching.
- **Deployment name phải match Service name:** Hoàn toàn sai — tên là độc lập. Service không liên kết với Deployment theo tên, mà liên kết với Pod theo label.

---

## Câu 4

**Which kubectl command should you use to view logs from a Pod that crashed and restarted?**

- `kubectl logs <pod-name> --previous` ✅
- `kubectl logs <pod-name>`
- `kubectl describe pod <pod-name>`

**Đáp án: `kubectl logs <pod-name> --previous`**

Khi Pod đang ở trạng thái `CrashLoopBackOff`, Pod đã restart nhiều lần. `kubectl logs <pod-name>` (không có flag) chỉ hiển thị log của **lần chạy hiện tại** — thường rất ít vì Pod crash rất nhanh.

`--previous` lấy log từ **container instance trước đó** — thường chứa error message thực sự gây ra crash.

```bash
kubectl logs inference-api-7d8b4f9c6-abc12 --previous
# Hiển thị log từ lần crash trước — thấy được error thực sự
```

Tại sao các đáp án còn lại sai:
- **`kubectl logs <pod-name>`** (không flag): Hiện log lần chạy hiện tại. Nếu Pod vừa crash và restart, log này rất ngắn và có thể không có error message hữu ích.
- **`kubectl describe pod`:** Hiện Pod metadata, events (scheduling, image pull...), và resource usage — nhưng không hiện application log. Hữu ích cho infrastructure issues (Pending, ImagePullBackOff) nhưng không cho application crashes.

---

## Câu 5

**What does the `replicas` field in a Deployment manifest control?**

- The number of Pod copies that should run simultaneously ✅
- The number of containers in each Pod
- The number of Services that can connect to the Deployment

**Đáp án: Số Pod copies chạy đồng thời**

```yaml
spec:
  replicas: 2    # Luôn duy trì 2 Pod chạy đồng thời
```

Deployment controller monitor liên tục và đảm bảo luôn có đúng số Pod đang chạy. Nếu 1 Pod crash → Deployment tự động tạo Pod mới để đạt đủ `replicas`.

Tại sao các đáp án còn lại sai:
- **Số containers trong Pod:** Được xác định bởi mảng `containers` trong `spec.template.spec`, không phải `replicas`. Mỗi Pod có thể có nhiều container (sidecar pattern).
- **Số Services kết nối:** Services kết nối với Pod thông qua label selector — hoàn toàn độc lập với `replicas`. Không có giới hạn số Service kết nối với Deployment.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Service type cho internet-facing app | LoadBalancer |
| 2 | Behavior khi resource requests vượt node capacity | Pod stays Pending |
| 3 | Điều kiện traffic routing giữa Deployment và Service | Pod labels match Service selector |
| 4 | Xem log của Pod đã crash | `kubectl logs --previous` |
| 5 | Ý nghĩa của `replicas` field | Số Pod copies chạy đồng thời |

---

*Module hoàn thành. Bài tiếp theo: AKS Module 2*

---

[← Bài 3](./aks-m1-bai3-deploy-applications.md) · [🏠 Mục lục](../README.md) · [Bài 1 →](./aks-m2-bai1-configmaps.md)
