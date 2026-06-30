# Bài 3 — Verify Service Connectivity and Endpoints

> Khoá: AI-200 · AKS — Monitor and troubleshoot applications on Azure Kubernetes Service

---

## Tại sao cần verify connectivity?

Pod healthy không đồng nghĩa với AI API reachable. Service misconfiguration có thể tạo ra **intermittent failure** khó diagnose:
- Selector label sai → Service có no endpoints
- Port/targetPort không align → connection refused
- Ingress rules sai → request không đến đúng Service

Bài này tập trung vào việc kiểm tra toàn bộ traffic path từ client đến Pod.

---

## Review Service types cho AI scenario

| Service type | Reachable từ | Dùng khi |
|---|---|---|
| **ClusterIP** | Trong cluster only | Internal AI service (vector DB, embedding API) |
| **LoadBalancer** | Internet + Azure LB | External-facing AI inference API |
| **NodePort** | Node IP + port | Dev/testing, ít dùng production |

AI API cần external access → **LoadBalancer** hoặc **Ingress controller**.

---

## Review Services qua Azure Portal

**Portal → AKS cluster → Kubernetes resources → Services and ingresses**

- Xem tất cả Service across namespaces, filter theo namespace
- External IP column cho LoadBalancer Service
- **Endpoints section** trong Service detail — nếu trống → no backing Pods
- Ingress resources: host rules, paths, backend Services, external address

---

## Validate Service-to-Pod Connectivity bằng kubectl

```bash
# Liệt kê Service
kubectl get service -n ai-workloads

# Chi tiết Service — selector, port, endpoints
kubectl describe service inference-api -n ai-workloads

# Xem EndpointSlices — Pod IPs thực sự nhận traffic
kubectl get endpointslices \
  -l kubernetes.io/service-name=inference-api \
  -n ai-workloads
```

**Checklist khi xem Service:**

- **Selector** match với Pod labels không?
- **`port`** (external) và **`targetPort`** (container) có đúng không?
- **EndpointSlices** có Pod IP không? Nếu trống → label mismatch → fix labels

Sau khi fix labels → apply → confirm EndpointSlices xuất hiện.

---

## Test AI Endpoint bằng Port-Forward

Port-forward cho phép send traffic từ **máy local** đến Service/Pod bên trong cluster. Không cần ingress hay external IP.

```bash
# Forward local port 8080 → Service port 80
kubectl port-forward service/inference-api 8080:80 -n ai-workloads
```

Trong terminal khác:

```bash
# Test inference endpoint
curl http://localhost:8080/api/inference

# Test health endpoint
curl http://localhost:8080/health
```

**Dùng port-forward khi:**
- Debug model endpoint mới trước khi expose externally
- Ingress chưa được configure
- Cần test nhanh mà không ảnh hưởng production traffic

Nếu `curl` trả về response → Service và Pod hoạt động đúng. Nếu lỗi → correlate với log và metrics từ bài 1.

---

## Verify External Connectivity

### Tìm external address qua Portal

Portal → Services and ingresses → LoadBalancer Service → cột External IP.

Ingress resources → cột Address → external hostname hoặc IP.

### Tìm external address qua kubectl

```bash
# External IP của Service
kubectl get service inference-api -n ai-workloads

# Ingress external address
kubectl get ingress -n ai-workloads
```

Sau khi có external IP/hostname, test từ client hoặc test environment:

```bash
curl http://<external-ip>/api/inference
```

Confirm request reach đúng AI endpoint và trả về successful response.

---

## Tổng hợp Monitoring + Troubleshoot Workflow

Kết hợp 3 bài thành workflow hoàn chỉnh:

```
Phát hiện issue (latency spike, error rate tăng)
    ↓
Bài 1 — Portal Monitoring tab: overview CPU/memory/errors
    ↓
Bài 1 — kubectl logs -f: stream log khi reproduce issue
    ↓
Bài 2 — kubectl describe pod: events, probe, env vars
    ↓
Bài 2 — kubectl describe service: selector, ports, endpoints
    ↓
Bài 3 — kubectl get endpointslices: xác nhận Pod IPs
    ↓
Bài 3 — kubectl port-forward: test endpoint isolated
    ↓
Fix: update manifest → kubectl apply → verify
```

---

## Bản chất bài này là gì?

**Một câu:** Verify connectivity = xác nhận toàn bộ traffic path hoạt động — `port-forward` để test isolated (không cần expose externally), `endpointslices` để xem ground truth của routing.

### So sánh với Docker và ACA

| Thao tác | Docker | ACA | AKS |
|---|---|---|---|
| Test endpoint nội bộ trước expose | `curl localhost:port` | `az containerapp logs show` (không test trực tiếp) | `kubectl port-forward` ✅ |
| Xem routing thực tế | `docker inspect network` | `az containerapp revision list` | `kubectl get endpointslices` |
| Tìm external IP | Host IP | `az containerapp show --query ...fqdn` | `kubectl get svc` (EXTERNAL-IP) |
| Test internal service | `docker network connect` + curl | Không có tool | `kubectl run debug --image=alpine` |

**Port-forward là công cụ debug quan trọng không có trong Docker hay ACA:** Cho phép gửi request từ máy local thẳng vào Service/Pod bên trong cluster, không cần internet exposure. Dùng khi: test model endpoint mới, debug ingress chưa setup, production service không muốn expose public.

**EndpointSlices = ground truth cho routing:** Service có Endpoints ≠ Pod healthy. Khi Service nói "no endpoints" mặc dù Pod running → đây là label mismatch. `endpointslices` cho thấy Pod IP nào đang thực sự nhận traffic — đây là bước xác nhận cuối cùng.

---

## Checklist ghi nhớ cho AI-200

- [ ] External AI API → **LoadBalancer** hoặc Ingress
- [ ] Internal service (vector DB, embedding) → **ClusterIP**
- [ ] `kubectl describe service` → xem selector, port, endpoints
- [ ] `kubectl get endpointslices -l kubernetes.io/service-name=<name>` → Pod IPs thực sự nhận traffic
- [ ] EndpointSlices trống → label mismatch → fix selector hoặc Pod labels
- [ ] `kubectl port-forward service/<name> <local-port>:<service-port>` → test local
- [ ] Port-forward dùng cho debug trước khi expose externally
- [ ] `kubectl get ingress -n <namespace>` → xem ingress external address
- [ ] Pair connectivity test với logs và metrics để thấy impact của config change

---

*Module Assessment tiếp theo*

---

[← Bài 2](./aks-m3-bai2-troubleshoot-pods-services.md) · [🏠 Mục lục](../README.md) · [Assessment →](./aks-m3-module-assessment.md)
