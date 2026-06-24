# Bài 5 — Optimize Container Resources and Scaling

> Khoá: AI-200 · ACA — Manage containers in Azure Container Apps

---

## Tổng quan

Resource sizing và scaling ảnh hưởng đến hai thứ cùng lúc: **performance** và **cost**. Với AI service, hai vấn đề phổ biến nhất là:

- **CPU thiếu** → throttling → latency spike (request chậm bất thường)
- **Memory thiếu** → OOM kill → container restart → cold start → load model lại

Bài này ngắn nhưng có khái niệm quan trọng: cost = per-replica sizing × số lượng replica.

> **Lưu ý:** Resource và scaling options phụ thuộc vào environment configuration. Validate constraints và defaults trong documentation hiện tại trước khi deploy production.

---

## Per-replica Sizing và Total Cost

ACA scale bằng cách **thêm hoặc bớt replica** (instance). Mỗi replica có CPU và memory riêng.

**Total cost = per-replica cost × số replica đang chạy**

Ví dụ:
```
Scenario A: 1 replica × 2 CPU = 2 CPU total
Scenario B: 2 replica × 1 CPU = 2 CPU total

Cùng total compute, nhưng:
- Scenario A: ít cold start hơn, 1 replica handle nhiều request concurrent
- Scenario B: fault tolerant hơn (nếu 1 replica crash, còn 1 replica kia)
```

### Đo lường trước khi optimize

Không đoán mò — **measure trước, optimize sau**:

| Workload | Cần đo | Dấu hiệu cần tăng resource |
|---|---|---|
| AI API (synchronous) | Latency dưới concurrent load | CPU throttling = tăng CPU |
| Background worker | Throughput (requests/giây) | OOM restart = tăng memory |
| Model inference | Time per request | Latency spike = tăng CPU hoặc dùng GPU |

---

## Cập nhật CPU và Memory

```bash
az containerapp update \
    --name ai-api \
    --resource-group rg-aca-demo \
    --cpu 1.0 \
    --memory 2Gi
```

`--cpu` tính bằng vCores (0.25, 0.5, 1.0, 2.0...). `--memory` theo format `<số>Gi` (0.5Gi, 1Gi, 2Gi...).

Sau khi update, ACA tạo revision mới với resource setting mới. Validate bằng cách:
1. Kiểm tra latency có cải thiện không
2. Kiểm tra revision stable dưới load (không OOM restart)

### CPU và Memory phải match nhau

ACA có quy định về ratio CPU:Memory. Không phải mọi combination đều hợp lệ. Ví dụ: 0.25 CPU thường đi với 0.5Gi memory. Validate combination trong documentation trước khi set.

---

## Scaling Strategy — API vs Background Worker

Hai loại workload cần scaling strategy khác nhau:

### Synchronous API — Cần stability và predictable latency

```yaml
# YAML scaling config cho API
properties:
  template:
    scale:
      minReplicas: 1      # Luôn có ít nhất 1 replica → không có cold start
      maxReplicas: 10     # Scale up đến tối đa 10 khi cần
```

**Tại sao cần `minReplicas: 1` cho AI API?**

Khi `minReplicas: 0` (scale-to-zero), lần đầu có traffic sau idle period phải:
1. Pull image (nếu chưa cache)
2. Start container
3. Load AI model vào memory (30-120 giây)
4. Pass readiness probe

User phải chờ tất cả bước trên — cold start với AI model có thể tốn hàng phút. `minReplicas: 1` tránh điều này bằng cách luôn giữ ít nhất 1 replica warm.

**Trade-off:** Minimum replica tăng baseline cost — 1 replica chạy 24/7 dù không có traffic.

### Background Worker — Có thể chấp nhận scale-to-zero

```yaml
# YAML scaling config cho background worker
properties:
  template:
    scale:
      minReplicas: 0      # Scale-to-zero khi không có work
      maxReplicas: 5
```

Background worker xử lý queue hoặc event — khi queue rỗng, không cần replica nào chạy. Scale-to-zero tiết kiệm cost khi workload intermittent.

**Yêu cầu:** Worker phải **drain work safely** khi scale down — không drop message đang xử lý giữa chừng.

---

## Alignment: Scaling + Probe Configuration

Đây là điểm hay bị bỏ sót — scaling và probe phải được cấu hình **phối hợp** với nhau.

**Scenario vấn đề:**

```
Rollout revision mới → ACA tạo replica mới
    ↓
Replica mới bắt đầu load AI model (45 giây)
    ↓
Readiness probe initialDelaySeconds = 10 giây
    ↓
Probe fail (model chưa load) → ACA không route traffic
    ↓
failureThreshold = 3 → fail 3 lần → revision marked Unhealthy
    ↓
Rollout trông như fail — nhưng thực ra model chỉ cần thêm thời gian
```

**Giải pháp:** Align probe `initialDelaySeconds` với thời gian load model. Nếu model mất 45 giây, set `initialDelaySeconds: 60` để probe không bắt đầu trước khi model ready.

---

## Best Practices — Optimize có hệ thống

### Size for the bottleneck — Tăng đúng resource

```
Thấy latency spike → check CPU throttling metrics
    → Có throttling? → tăng CPU
    → Không throttling? → vấn đề ở chỗ khác (network, dependency...)

Thấy container restart unexpected → check log có OOMKilled không
    → Có OOM? → tăng memory
    → Không OOM? → check liveness probe quá aggressive
```

### Thay đổi một biến một lúc

Đừng vừa tăng CPU vừa tăng memory vừa thay đổi scaling config trong cùng một update. Nếu performance cải thiện, bạn không biết variable nào có tác dụng. Nếu performance tệ hơn, bạn không biết nguyên nhân.

### Reassess sau mỗi lần update model

Model version mới có thể thay đổi:
- **Startup time** → cần điều chỉnh probe `initialDelaySeconds`
- **Memory usage** → model lớn hơn cần nhiều memory hơn
- **Request latency** → model mới có thể nhanh hơn hoặc chậm hơn → ảnh hưởng sizing

Không assume resource setting từ model cũ vẫn đúng cho model mới.

### Minimum replicas — Cost vs cold start trade-off

```
minReplicas = 0  → Zero baseline cost, nhưng cold start khi có traffic sau idle
minReplicas = 1  → Baseline cost 1 replica 24/7, nhưng không cold start
minReplicas = 2  → Fault tolerant + không cold start, nhưng cost cao hơn
```

Với production AI API phục vụ user interactive → `minReplicas: 1` thường là lựa chọn đúng.
Với dev/staging AI API không cần SLA → `minReplicas: 0` để tiết kiệm cost.

---

## Bản chất bài này là gì?

**Một câu:** Resource sizing trong ACA = per-replica config × replica count = total capacity; cả hai chiều đều tunable, không như App Service (plan = fixed tier).

### So sánh

| | Docker | App Service | ACA |
|---|---|---|---|
| CPU/Memory per container | `--cpus --memory` (runtime) | Tiers (S1, S2...) cố định | `--cpu --memory` trên mỗi revision |
| Scale replica | Manual hoặc Swarm | Manual hoặc Auto Scale | Auto scale rules |
| Đơn vị tính tiền | Bạn trả cho server | Bạn trả cho plan tier (luôn chạy) | Pay per replica per time |
| Scale-to-zero cost | Không áp dụng (server vẫn chạy) | ❌ Free/Shared tier chỉ | ✅ Zero cost khi 0 replica |

**Trade-off ít được nhắc đến:** Replica nhỏ → scale granularity tốt hơn nhưng overhead mỗi lần scale event. Replica lớn → ít scale event nhưng cost nhảy theo bậc lớn hơn. Không có "đúng hoàn toàn" — depends on traffic pattern.

---

## Checklist ghi nhớ cho AI-200

- [ ] **Total cost = per-replica cost × số replica** — tune cả hai chiều
- [ ] CPU thiếu → **throttling** → latency spike
- [ ] Memory thiếu → **OOM kill** → restart → cold start → load model lại
- [ ] Measure trước: API đo latency dưới concurrency · Worker đo throughput
- [ ] Update resource: `az containerapp update --cpu <cores> --memory <size>Gi`
- [ ] Update tạo **revision mới** — validate revision stable dưới load sau đó
- [ ] CPU và Memory phải theo **ratio hợp lệ** — validate với documentation
- [ ] **Synchronous AI API** → `minReplicas: 1` để tránh cold start
- [ ] **Background worker** → `minReplicas: 0` nếu chấp nhận cold start và drain safely
- [ ] Scaling và probe phải **align** — `initialDelaySeconds` phải đủ lớn cho model load
- [ ] Thay đổi **một biến một lúc** khi optimize
- [ ] **Reassess sau model update** — startup time, memory, latency có thể thay đổi

---

*Bài tiếp theo: Exercise — Diagnose and fix a failing deployment*
