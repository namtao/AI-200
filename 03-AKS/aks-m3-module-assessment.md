# Module Assessment — Monitor and Troubleshoot Applications on Azure Kubernetes Service

> Khoá: AI-200 · AKS — Monitor and troubleshoot applications on Azure Kubernetes Service

---

## Câu 1

**You receive reports that an AI inference API on Azure Kubernetes Service occasionally returns HTTP 500 errors and higher latency. You want to inspect recent error messages for a specific pod while reproducing the issue. Which approach is most appropriate?**

- Use `kubectl logs -f <pod-name> -n <namespace>` while you send test requests to the API. ✅
- Open Azure Monitor and review only node CPU metrics for the last week.
- Run `kubectl describe node` on all nodes to look for scheduling events.

**Đáp án: `kubectl logs -f` trong khi reproduce**

`-f` (follow) stream log real-time — bạn thấy error message ngay khi request fail. Đây là cách nhanh nhất để capture error context đúng lúc nó xảy ra.

```bash
kubectl logs -f <pod-name> -n ai-workloads
# Trong terminal khác: gửi test request để trigger lỗi
curl http://<service-ip>/api/inference
```

Tại sao các đáp án còn lại sai:
- **Azure Monitor node CPU metrics for last week:** HTTP 500 errors là application-level error — không thể thấy trong node CPU metrics. Metrics từ tuần trước cũng không giúp reproduce và capture error hiện tại. Sai cả tool lẫn thời gian.
- **`kubectl describe node` cho scheduling events:** Node scheduling events là về infrastructure — Pod nào được schedule trên node nào. Không liên quan đến HTTP 500 response của AI API.

---

## Câu 2

**A pod that runs a model-serving container is stuck in CrashLoopBackOff. You want to understand why the container exits. What should you do first?**

- Run `kubectl describe pod <pod-name> -n <namespace>` and inspect events and container status. ✅
- Immediately delete the pod so Kubernetes recreates it.
- Scale the Deployment to zero replicas and then scale it back up.

**Đáp án: `kubectl describe pod` và inspect events**

`kubectl describe pod` hiển thị:
- **Events** — image pull fail, probe fail, scheduling issue, OOMKill
- **Container state** — exit code của lần crash (137 = OOMKill, khác = app error)
- **Environment variables** — missing config?
- **Volume mounts** — model files có được mount không?

Từ thông tin đó mới biết cần xem log (`kubectl logs --previous`) để thấy error message.

Tại sao các đáp án còn lại sai:
- **Delete Pod:** Kubernetes sẽ recreate Pod mới ngay — Pod tiếp tục CrashLoopBackOff vì nguyên nhân chưa được fix. Xoá Pod không giải quyết vấn đề, chỉ clear evidence tạm thời.
- **Scale to zero rồi scale up:** Tương tự — restart Pod mà không biết nguyên nhân. Pod mới sẽ crash với cùng lý do. Đây là workaround mù quáng, không phải troubleshoot.

---

## Câu 3

**A Service that fronts your AI API shows no endpoints, even though the pods appear healthy and ready. Which command helps you confirm whether the Service selectors match pod labels?**

- `kubectl describe service <service-name> -n <namespace>` ✅
- `kubectl top nodes`
- `kubectl logs <pod-name> -n <namespace>`

**Đáp án: `kubectl describe service`**

```bash
kubectl describe service inference-api -n ai-workloads
```

Output hiển thị:
- **Selector** — label selector của Service
- **Endpoints** — `<none>` confirms không có Pod match
- **Port/targetPort** — có align không

So sánh Selector với Pod labels (`kubectl get pods --show-labels`) để tìm mismatch. Đây là nguyên nhân phổ biến nhất khi Service có no endpoints.

Tại sao các đáp án còn lại sai:
- **`kubectl top nodes`:** Hiển thị CPU/memory usage của node — không liên quan đến Service selector hay Pod labels. Dùng để check resource pressure, không phải connectivity.
- **`kubectl logs`:** Hiển thị application log từ Pod — Pod đang healthy nên log không giúp gì để hiểu tại sao Service không route traffic. Vấn đề là configuration mismatch, không phải application error.

---

## Câu 4

**You need to debug a new AI endpoint inside the cluster before exposing it externally. You want to send HTTP requests from your development machine directly to the Service. Which command should you use?**

- `kubectl port-forward service/<service-name> 8080:80 -n <namespace>` ✅
- `kubectl get endpoints <service-name> -n <namespace>`
- `kubectl describe node` on the node that hosts the pod

**Đáp án: `kubectl port-forward`**

```bash
kubectl port-forward service/inference-api 8080:80 -n ai-workloads
```

Lệnh này forward port 8080 trên máy local đến port 80 của Service bên trong cluster. Không cần external IP hay ingress. Từ terminal khác:

```bash
curl http://localhost:8080/api/inference
```

Nếu response thành công → Service và Pod hoạt động đúng. Debug endpoint mới mà không ảnh hưởng production.

Tại sao các đáp án còn lại sai:
- **`kubectl get endpoints`:** Hiện Pod IPs đang backing Service — cho biết Service có endpoints không, nhưng không cho bạn **gửi HTTP request** đến endpoint. Đây là read-only inspection, không phải testing tool.
- **`kubectl describe node`:** Node information — không liên quan gì đến testing AI endpoint hay forward traffic từ local machine.

---

## Câu 5

**Metrics for a model-serving pod show sustained CPU usage at its configured limit, and users report increased latency. What is the most appropriate next step?**

- Adjust CPU requests and limits or scale out replicas so the pod has enough capacity. ✅
- Ignore the metrics because the pod is still running.
- Delete the Service and recreate it with the same configuration.

**Đáp án: Adjust CPU resources hoặc scale out**

CPU đạt limit → Kubernetes **throttle** Pod (không terminate) → inference request xử lý chậm hơn → latency tăng. Correlation rõ ràng: CPU throttle = latency spike.

Hai hướng fix:
```bash
# Option 1: Tăng CPU limit cho Pod
kubectl set resources deployment inference-api --limits=cpu=2000m -n ai-workloads

# Option 2: Scale out thêm replica
kubectl scale deployment inference-api --replicas=3 -n ai-workloads
```

Sau khi adjust → monitor `kubectl top pods` để confirm CPU usage không còn ở limit.

Tại sao các đáp án còn lại sai:
- **Ignore metrics:** CPU throttle trực tiếp gây latency tăng — đây là correlation đã được confirm qua metrics + user report. Bỏ qua là inappropriate vì vấn đề đang ảnh hưởng user experience.
- **Delete và recreate Service:** Service là networking layer — không ảnh hưởng đến CPU usage của Pod. Recreate Service với cùng config hoàn toàn không giải quyết CPU throttle. Đây là wrong layer.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Inspect log khi reproduce HTTP 500 | `kubectl logs -f` (follow real-time) |
| 2 | Hiểu tại sao container crash (CrashLoopBackOff) | `kubectl describe pod` + inspect events |
| 3 | Confirm Service selector match Pod labels | `kubectl describe service` |
| 4 | Test AI endpoint từ local trước khi expose external | `kubectl port-forward service/...` |
| 5 | Fix CPU throttle gây latency tăng | Tăng CPU limit hoặc scale out replicas |

---

*Module hoàn thành. AKS Learning Path kết thúc.*

---

[← Bài 3](./aks-m3-bai3-verify-connectivity.md) · [🏠 Mục lục](../README.md) · [Tổng hợp →](./aks-summary.md)
