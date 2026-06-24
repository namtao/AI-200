# Bài 2 — Expose Applications in Azure Kubernetes Service

> Khoá: AI-200 · AKS — Deploy applications to Azure Kubernetes Service

---

## Tại sao cần Service?

Pod có IP address riêng bên trong cluster, nhưng IP này **thay đổi mỗi khi Pod restart**. Nếu client kết nối trực tiếp vào Pod IP → mỗi lần Pod crash và restart, client phải biết IP mới.

**Service** giải quyết vấn đề này bằng cách tạo **stable endpoint** (IP cố định hoặc DNS name) và tự động route traffic đến Pod đang healthy.

```
Client → Service (stable IP) → Pod A
                             → Pod B (load balanced)
```

Khi Pod A crash và restart với IP mới → Service tự cập nhật, client không cần biết.

---

## 4 Service Types

### ClusterIP (default)

**Internal only** — chỉ accessible trong cluster. Pod trong cluster dùng DNS name để gọi nhau.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: vector-db-service
spec:
  type: ClusterIP           # Có thể bỏ qua — ClusterIP là default
  selector:
    app: vector-db
  ports:
  - port: 6333
    targetPort: 6333
```

**Dùng khi:** Service nội bộ không cần external access — vector database, embedding service, internal cache, backend service chỉ được gọi bởi service khác trong cluster.

**Truy cập từ Pod khác:** `http://vector-db-service.default.svc.cluster.local:6333`

### NodePort

**Expose trên port của tất cả node**. External client truy cập bằng `node-ip:nodeport`.

**Dùng khi:** Development/testing, muốn external access đơn giản mà không cần Azure Load Balancer.

NodePort tự động assign port trong range 30000-32767.

### LoadBalancer ⭐

**Tạo Azure Load Balancer với public IP**. Traffic từ internet đến public IP → Load Balancer → Service → Pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: inference-api-service
spec:
  type: LoadBalancer              # External load balancer với public IP
  selector:
    app: inference-api            # Route đến Pod có label này
  ports:
  - protocol: TCP
    port: 80                      # Port client kết nối (external)
    targetPort: 8080              # Port app lắng nghe trong container
```

**Dùng khi:** Production AI API cần internet access, phục vụ external user.

**Azure tự động:** Provision Azure Load Balancer, assign public IP, configure health probe.

### ExternalName

Map Service đến external DNS name — ít dùng cho deployment thông thường.

---

## Port Mapping — Quan trọng cần hiểu rõ

```yaml
ports:
- protocol: TCP
  port: 80          # ← Client kết nối port này (external)
  targetPort: 8080  # ← App lắng nghe port này (bên trong container)
```

**Luồng traffic:**

```
Client → port 80 (Service) → port 8080 (Container)
```

Ví dụ: FastAPI app chạy trên port 8080 bên trong container. Client gọi port 80 (standard HTTP). Service forward từ 80 → 8080.

**Common targetPort:**
- Flask/FastAPI: 5000 hoặc 8000 hoặc 8080
- Node.js Express: 3000
- Spring Boot: 8080
- ASP.NET Core: 80

---

## Service-to-Pod Connection — Selector là chìa khóa

Service tìm Pod để route traffic thông qua **selector labels**:

```yaml
# Trong Deployment template — Pod nhận label này
template:
  metadata:
    labels:
      app: inference-api     # ← Pod có label "app: inference-api"

# Trong Service — route đến Pod có label này
selector:
  app: inference-api         # ← Phải khớp với Pod label
```

**Nếu selector không khớp:** Service có no endpoints → traffic không đến đâu cả. Đây là lỗi phổ biến nhất khi setup Service.

Kubernetes tự động cập nhật danh sách Pod phía sau Service khi Pod được tạo/xoá/crash.

---

## Truy cập Service

### ClusterIP — Internal DNS

Từ Pod khác trong cluster:
```
http://inference-api-service.default.svc.cluster.local:8080
         ↑service name    ↑namespace  ↑cluster suffix  ↑port
```

Viết ngắn nếu trong cùng namespace: `http://inference-api-service:8080`

### NodePort — Node IP + Port

```bash
kubectl get svc
# Output: PORT(S) → 80:30523/TCP
# NodePort là 30523

# Truy cập qua node IP
http://192.168.1.10:30523
```

### LoadBalancer — Public IP

```bash
kubectl get svc inference-api-service
# EXTERNAL-IP: 40.89.123.45

http://40.89.123.45
```

Nếu EXTERNAL-IP là `<pending>` → Azure đang provision Load Balancer, chờ vài phút.

---

## Bản chất bài này là gì?

**Một câu:** Kubernetes Service giải quyết vấn đề Pod IP thay đổi — nó là layer "name resolution + load balancing" ổn định trước Pod fleet có thể đến và đi bất kỳ lúc nào.

### So sánh với Docker và các giải pháp khác

| Giải pháp | Expose cách nào | Tương đương AKS |
|---|---|---|
| `docker run -p 80:8080` | Port mapping trực tiếp trên host | NodePort (không có LB) |
| Docker Compose + ports | Chỉ accessible trên host | Không phải production |
| Nginx upstream | Manual config, không auto-update | Service (auto-update endpoints) |
| Azure Load Balancer thủ công | Tạo bằng tay, point đến VM | Service type: LoadBalancer (AKS tự provision) |
| ACA ingress external/internal | Managed, per-app | LoadBalancer / ClusterIP |

**3 Service types cần nhớ cho AI-200:**
- `ClusterIP` = internal service (vector DB, embedding API, cache) — không accessible từ internet
- `LoadBalancer` = production external AI API — Azure provision LB + public IP tự động
- `NodePort` = dev/testing only — không dùng production

**Selector-based routing không có trong Docker:** Docker không có khái niệm "find containers matching label X". Kubernetes Service tự động cập nhật danh sách Pod khi Pod mới join/leave — không cần manual config. Lỗi selector mismatch → Service có no endpoints → silent routing failure.

---

## Checklist ghi nhớ cho AI-200

- [ ] Service tạo **stable endpoint** vì Pod IP thay đổi khi restart
- [ ] **ClusterIP** = internal only · **NodePort** = node IP + port · **LoadBalancer** = public IP · **ExternalName** = map DNS
- [ ] Production AI API → **LoadBalancer**
- [ ] Internal service (vector DB, embedding service) → **ClusterIP**
- [ ] `port` = external port client kết nối · `targetPort` = container port app lắng nghe
- [ ] **Selector phải match Pod labels** — nếu không khớp → no endpoints → traffic không route được
- [ ] ClusterIP DNS: `servicename.namespace.svc.cluster.local`
- [ ] LoadBalancer EXTERNAL-IP `<pending>` = Azure đang provision, chờ thêm

---

*Bài tiếp theo: Bài 3 — Deploy applications to Azure Kubernetes Service*
