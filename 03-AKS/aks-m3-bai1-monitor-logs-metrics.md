# Bài 1 — Monitor Application Logs and Metrics

> Khoá: AI-200 · AKS — Monitor and troubleshoot applications on Azure Kubernetes Service

---

## Tại sao monitoring quan trọng với AI workload?

AI application có đặc thù riêng cần monitor:
- **Latency spike** → có thể do model issue hoặc resource saturation
- **Error rate tăng** → upstream model endpoint fail hoặc config sai
- **Pod restart** → memory leak, OOMKill, unhandled exception
- **CPU saturation** → inference workload cần nhiều tài nguyên hơn allocated

Monitoring giúp bạn phân biệt **normal variation** với **emerging incident** — quyết định khi nào scale, investigate, hay rollback.

---

## Hai công cụ chính

AKS hỗ trợ monitoring qua hai luồng song song:

| | Azure Portal | kubectl |
|---|---|---|
| Phù hợp | Visual overview nhanh, chia sẻ với team | Targeted query, scripting, filter output |
| Logs | Live Logs streaming trên browser | `kubectl logs` |
| Metrics | Monitoring tab, Container insights dashboard | `kubectl top` |
| Không cần | Cài thêm gì | kubectl configured |

**Workflow điển hình:** Portal cho overview → kubectl cho investigation chi tiết.

---

## Key Monitoring Signals cho AI workload

- **Response latency và throughput** của AI endpoint
- **Error rate** — HTTP 5xx, timeout
- **Pod restart count** và container exit code
- **CPU và memory utilization** so với requests/limits đã config

---

## Xem Logs qua Azure Portal

**Portal → AKS cluster → Kubernetes resources → Workloads → chọn Pod → Live Logs**

Live Logs stream stdout/stderr real-time trong browser. Tính năng:
- Pause stream, search text
- Switch giữa container trong multi-container Pod
- Chia sẻ browser session với colleague

**Portal → Monitoring → Insights** → Container insights với unified view qua toàn cluster, filter theo namespace/Pod/container.

---

## Xem Logs bằng kubectl

```bash
# Liệt kê Pod trong namespace
kubectl get pods -n ai-workloads

# Xem log của Pod
kubectl logs <pod-name> -n ai-workloads

# Follow log real-time
kubectl logs -f <pod-name> -n ai-workloads

# Pod có nhiều container — chỉ định container
kubectl logs <pod-name> -c inference-api -n ai-workloads

# Log từ tất cả Pod có label app=inference-api
kubectl logs -l app=inference-api -n ai-workloads
```

**Namespace** là logical boundary nhóm workload liên quan. Flag `-n ai-workloads` chỉ kubectl nhìn vào namespace `ai-workloads`. Bỏ flag nếu workload ở `default` namespace.

**Namespaces vs Labels:**
- Namespace → tách environment, kiểm soát access/quotas
- Label → select Pod cụ thể trong namespace để scale, rollout, filter log

---

## Xem Resource Metrics qua Azure Portal

**Portal → AKS cluster → Monitoring tab** → CPU và memory graphs theo node pool.

**Portal → Monitoring → Insights → Nodes/Controllers/Containers tab** → heat maps, performance charts highlight Pod/node bị resource constraint.

**Live Metrics cho Pod:**
Portal → Monitoring → Insights → chọn Pod → Live Metrics → real-time CPU, memory, network.

---

## Xem Resource Metrics bằng kubectl

```bash
# Metrics per node
kubectl top nodes

# Metrics per pod trong namespace
kubectl top pods -n ai-workloads
```

Cần **metrics server** được cài trong cluster. So sánh output với `requests` và `limits` trong Pod spec:
- Pod liên tục đạt CPU limit → bị throttle → inference latency tăng
- Pod gần memory limit → nguy cơ OOMKill

---

## Structured Logging cho AI Service

Log nên bao gồm:
- **Request ID / Correlation ID** — trace request qua nhiều service
- **Revision/build ID** — biết đang chạy version nào
- **Model version** — model nào phục vụ request này
- **Latency breakdown** — total time + timing của từng dependency

**Structured format** (JSON) dễ query hơn trong Container insights và kubectl.

---

## Bản chất bài này là gì?

**Một câu:** AKS monitoring là two-tool setup — Portal cho visual overview và team sharing, kubectl cho targeted investigation — tương tự Docker nhưng thêm namespace và label để filter trong cluster nhiều app.

### So sánh với Docker và ACA

| Thao tác | Docker | ACA | AKS |
|---|---|---|---|
| Xem log | `docker logs container` | `az containerapp logs show` | `kubectl logs pod-name -n ns` |
| Log real-time | `docker logs -f container` | `--follow --tail` | `kubectl logs -f pod-name` |
| Nhiều container | `docker logs service` (compose) | `az containerapp logs show` | `kubectl logs -l app=label` |
| Resource usage | `docker stats` | Azure Monitor | `kubectl top pods` (cần metrics server) |
| Visual dashboard | Docker Desktop | Azure Portal | Azure Portal Container insights |
| Filter theo env | Compose project | Revision/Replica | Namespace + label |

**Namespace = logical isolation trong cluster:** Một AKS cluster có thể chạy `production` và `staging` trong các namespace khác nhau — kubectl mặc định xem namespace `default`. Phải thêm `-n <namespace>` để xem workload ở namespace khác.

**Structured logging là bắt buộc với AI service:** Không có shell access như App Service Kudu. Nếu log không có Correlation ID, Model version, Latency breakdown → khi incident xảy ra, không thể trace nguyên nhân.

---

## Checklist ghi nhớ cho AI-200

- [ ] 4 key signals: latency, error rate, pod restarts, CPU/memory vs limits
- [ ] Portal: **Live Logs** (real-time) · **Container insights** (historical, cross-pod)
- [ ] `kubectl logs -f` = follow real-time · `kubectl logs -l <label>` = tất cả Pod cùng label
- [ ] `-n <namespace>` flag để chỉ định namespace
- [ ] `kubectl top nodes` và `kubectl top pods` cần **metrics server**
- [ ] CPU throttle = latency tăng · Memory gần limit = OOMKill risk
- [ ] Workflow: Portal overview → kubectl investigation
- [ ] Log nên có: correlation ID, model version, latency breakdown

---

*Bài tiếp theo: Bài 2 — Troubleshoot Pods and Services*
