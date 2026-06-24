# Bài 2 — Troubleshoot Pods and Services

> Khoá: AI-200 · AKS — Monitor and troubleshoot applications on Azure Kubernetes Service

---

## Pod States cần nhận biết

| State | Ý nghĩa | Hướng xử lý |
|---|---|---|
| `ImagePullBackOff` | Không pull được image (sai path/tag hoặc không có credentials) | Verify image reference, check registry access |
| `CrashLoopBackOff` | Container crash liên tục | Xem log `--previous`, check env vars và secrets |
| `Pending` | Không schedule được | Xem events — insufficient CPU/memory hoặc node issue |
| Nhiều restarts | Memory leak hoặc unhandled exception | Xem log, check memory limits |

---

## Inspect Pod bằng Azure Portal

**Portal → AKS cluster → Kubernetes resources → Workloads**

- Xem Pod status, restart count, age ở dạng list
- Filter theo namespace
- Live Logs stream log ngay trong browser

**Portal → AKS cluster → Diagnose and solve problems**

Tool guided troubleshooting — chạy diagnostics tự động cho connectivity, node health, cluster config. Suggest remediation steps. Tốt cho người mới hoặc khi chưa rõ vấn đề ở đâu.

---

## Inspect Pod bằng kubectl describe

```bash
# Liệt kê Pod
kubectl get pods -n ai-workloads

# Xem chi tiết Pod và events
kubectl describe pod <pod-name> -n ai-workloads

# Xem events toàn namespace
kubectl get events -n ai-workloads
```

**Trong output của `kubectl describe pod`, chú ý:**

- **Events section** → failed image pull, scheduling issue, probe failure
- **Readiness/Liveness probe** → đang configured đúng chưa?
- **Environment variables** → đúng chưa? Secret reference có valid không?
- **Volume mounts** → model files hay config có được mount đúng không?
- **Container state** → exit code khi crash (OOMKill = 137, manual kill = 143...)

---

## Debug bên trong Container

Đôi khi cần vào trong container để kiểm tra môi trường thực tế.

### Qua Azure Portal

**Portal → AKS cluster → Workloads → chọn Pod → Console tab**

Mở terminal session trực tiếp trên browser. Không cần kubectl configured locally.

### Qua kubectl exec

```bash
kubectl exec -it <pod-name> -n ai-workloads -- /bin/sh
```

Từ bên trong container, có thể:

```bash
# Liệt kê model files hoặc config
ls /app/models/

# Kiểm tra env vars
env | grep API_KEY
printenv MODEL_ENDPOINT

# Test endpoint local
curl http://localhost:8080/health

# Kiểm tra dependencies có reachable không
curl http://vector-db-service:6333/health
```

> **Quan trọng:** Chỉ dùng để điều tra, không sửa trực tiếp trong container. Thay đổi phải đi qua manifest và redeploy để môi trường AKS reproducible.

---

## Inspect Service bằng Azure Portal

**Portal → AKS cluster → Kubernetes resources → Services and ingresses**

Xem Service details: type, cluster IP, external IP, port mappings, và **endpoint list** (Pod nào đang nhận traffic). Nếu endpoint list trống → Service không route được traffic dù Pod healthy.

---

## Inspect Service bằng kubectl

```bash
# Liệt kê tất cả Service
kubectl get service -n ai-workloads

# Chi tiết Service — xem selector, port, endpoints
kubectl describe service <service-name> -n ai-workloads

# Xem EndpointSlices — danh sách Pod IP thực sự nhận traffic
kubectl get endpointslices -l kubernetes.io/service-name=<service-name> -n ai-workloads
```

**Trong output `kubectl describe service`, kiểm tra:**

- **Selector** → có match với Pod labels không?
- **Port/targetPort** → có align với container port không?
- **Endpoints** → có Pod IP nào không? Nếu `<none>` → label mismatch

---

## Structured Troubleshooting Workflow

```
1. kubectl get pods -n ai-workloads
   → Identify Pod state (Running/CrashLoopBackOff/Pending...)
         ↓
2. kubectl describe pod <pod-name>
   → Xem Events: image pull fail? probe fail? scheduling issue?
         ↓
3. kubectl logs <pod-name> --previous
   → Xem lý do crash (missing env var, config error, exception...)
         ↓
4. kubectl exec -it <pod-name> -- /bin/sh
   → Verify runtime environment (env vars, files, connectivity)
         ↓
5. kubectl describe service <service-name>
   → Verify selector match, ports, endpoints
         ↓
6. kubectl get endpointslices -l kubernetes.io/service-name=<name>
   → Confirm Pod IPs trong EndpointSlices
```

---

## Bản chất bài này là gì?

**Một câu:** AKS troubleshooting = có shell access đầy đủ vào container (`kubectl exec`) nhưng quy tắc vàng là "diagnose only, fix through manifest" — khác App Service thiếu shell, giống Docker exec nhưng có thêm `describe` để xem events K8s.

### So sánh với Docker và ACA

| Thao tác | Docker | ACA | AKS |
|---|---|---|---|
| Xem trạng thái app | `docker ps` | `az containerapp revision list` | `kubectl get pods` |
| Debug chi tiết | `docker inspect` | ❌ Không có | `kubectl describe pod` + Events |
| Vào trong container | `docker exec -it container bash` | ❌ Không có | `kubectl exec -it pod -- /bin/sh` |
| Xem log crash | `docker logs --previous` | Log by revision | `kubectl logs --previous` |
| Log toàn bộ service | `docker logs service` | N/A | `kubectl logs -l app=label` |
| Tự động detect vấn đề | ❌ | ❌ | Portal "Diagnose and solve" ✅ |

**Exit code quan trọng:** OOMKill = 137. App error thường = 1 hoặc non-zero khác. Biết exit code → biết nguyên nhân ngay.

**"Diagnose in container, fix in manifest":** Kubernetes tạo ra reprodicibility — container bị xoá và recreate theo manifest. Fix trực tiếp trong container sẽ bị mất khi Pod restart. Luôn cập nhật manifest rồi `kubectl apply` để fix persist.

**Workflow 6 bước:** `get pods` → `describe pod` → `logs --previous` → `exec` → `describe service` → `get endpointslices`. Thứ tự này đi từ ngoài vào trong, từ infrastructure đến application.

---

## Checklist ghi nhớ cho AI-200

- [ ] Portal **Diagnose and solve problems** → auto-detect common issues
- [ ] `kubectl describe pod` → Events section, probe config, env vars, mounts
- [ ] `kubectl get events -n <namespace>` → events toàn namespace
- [ ] Exit code 137 = OOMKill, exit code khác = app error
- [ ] `kubectl exec -it <pod> -- /bin/sh` → debug bên trong container
- [ ] Chỉ diagnose, không fix in-container — thay đổi qua manifest
- [ ] Service có **no endpoints** → selector không match Pod labels
- [ ] `kubectl get endpointslices` → xem Pod IP thực sự nhận traffic
- [ ] Portal Console tab = exec shell qua browser (không cần kubectl)

---

*Bài tiếp theo: Bài 3 — Verify Service Connectivity and Endpoints*
