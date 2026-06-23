# Bài 4 — Select Compute Resources for Performance and Cost

> Khoá: AI-200 · ACA — Scale containers in Azure Container Apps

---

## Resource Allocation — Cơ bản

ACA allocate CPU và memory **per container** (không phải per replica). Mỗi replica nhận đúng resources đã config.

**Đơn vị:** CPU tính bằng cores (hoặc fraction), memory tính bằng GiB.

**Default:** 0.25 CPU cores + 0.5 GiB memory — phù hợp lightweight app.

**Constraint quan trọng: Memory ≥ CPU × 2**

```
CPU 0.25 → minimum memory 0.5 GiB
CPU 0.5  → minimum memory 1.0 GiB
CPU 1.0  → minimum memory 2.0 GiB
CPU 4.0  → minimum memory 8.0 GiB (max Consumption plan)
```

**Maximum (Consumption plan):** 4 CPU cores, 8 GiB memory per container.

---

## CPU vs Memory — Behavior khi vượt limit

| Resource | Khi vượt limit | Hậu quả |
|---|---|---|
| **Memory** | Platform **terminate và restart** replica | Hard failure — OOM kill |
| **CPU** | Platform **throttle** replica (không terminate) | Performance degradation — latency spike |

→ Memory over-limit = crash. CPU over-limit = slow. Cả hai đều nguy hiểm nhưng khác nhau.

---

## Cấu hình Resources

```bash
az containerapp create \
  --name order-api \
  --resource-group rg-ecommerce \
  --environment my-environment \
  --image myregistry.azurecr.io/order-api:v1 \
  --cpu 0.5 \
  --memory 1.0Gi \
  --min-replicas 2 \
  --max-replicas 20
```

**Total capacity = per-replica CPU × max replicas**

Ví dụ: 0.5 CPU × 20 replicas = 10 total CPU cores tối đa.

### Multi-container app

Nếu app có nhiều container (main + sidecar), mỗi container có resource allocation riêng. Tổng cộng các container count against environment limits.

---

## Workload Profiles — Consumption vs Dedicated

### Consumption-only environment

- Serverless billing: pay chỉ khi replica chạy
- Share compute resources với tenants khác
- Max 4 CPU, 8 GiB per container
- Phù hợp: variable workload, scale-to-zero mang lại savings lớn

### Workload Profiles environment

Có thêm tùy chọn:

**Consumption profile** trong Workload profiles env: Tương tự Consumption-only về billing.

**Dedicated profiles:** VM sizes reserved cho workload của bạn — không share với tenant khác.

| Tính năng | Cần Dedicated |
|---|---|
| GPU workload | ✅ |
| > 4 CPU hoặc > 8 GiB memory | ✅ |
| Consistent performance (no variability) | ✅ |
| Compliance: dedicated infrastructure | ✅ |
| Variable workload, tiết kiệm cost | ❌ Dùng Consumption |

---

## Cost Optimization

### Scale-to-zero — Hiệu quả nhất cho intermittent workload

```yaml
# Background worker — scale về 0 khi không có message
scale:
  minReplicas: 0    # Zero cost khi idle
  maxReplicas: 20
```

Kết hợp với event-driven trigger (Service Bus, Event Hubs) để đảm bảo auto scale-up khi có work.

### Right-sizing

**Quy trình đúng:**
1. Deploy với default (0.25 CPU, 0.5 GiB)
2. Monitor actual usage qua Azure Monitor
3. Chỉ tăng khi thấy throttling (CPU) hoặc OOM kill (memory)

Đừng pre-optimize dựa trên đoán mò.

### Billing model

- **Idle replicas:** Billing ở idle rate (thấp hơn active rate)
- **Active replicas:** Billing ở active rate
- **Zero replicas:** Không tính compute charges

---

## Performance Optimization

### Minimum replicas để tránh cold start

```yaml
# Production AI API — luôn có ít nhất 1 replica
scale:
  minReplicas: 1    # Không bao giờ cold start
  maxReplicas: 10
```

Cold start với AI service = load model lại = 30-120 giây latency.

### Dedicated profiles cho consistent performance

Consumption plan có performance variability vì shared infrastructure. Nếu SLA yêu cầu consistent latency → Dedicated profile.

---

## Trade-off: Replica Size vs Scaling Granularity

| | Larger replicas (nhiều CPU/memory) | Smaller replicas |
|---|---|---|
| Scaling events | Ít hơn | Nhiều hơn |
| Cost granularity | Thô hơn (over-provision giữa steps) | Tinh hơn |
| Scaling overhead | Thấp hơn | Cao hơn |
| Phù hợp | Stable workload | Variable, bursty workload |

**Maximum replica count:** 1000 per revision.

---

## Checklist ghi nhớ cho AI-200

- [ ] Memory allocation phải **≥ CPU × 2 GiB**
- [ ] Default: **0.25 CPU, 0.5 GiB** — start đây rồi tăng khi cần
- [ ] Max Consumption plan: **4 CPU, 8 GiB** per container
- [ ] CPU over-limit → **throttle** (slow) · Memory over-limit → **terminate + restart**
- [ ] Multi-container: mỗi container có resource riêng, tổng count against env limits
- [ ] **Consumption-only:** Serverless, shared, max 4 CPU/8 GiB
- [ ] **Dedicated profile:** GPU, >4 CPU/8 GiB, consistent latency, compliance
- [ ] **Scale-to-zero** = most effective cost optimization cho intermittent workload
- [ ] Right-size: start default → monitor → tăng khi có evidence (throttling/OOM)
- [ ] `minReplicas: 1` cho production AI API để tránh cold start
- [ ] Max **1000 replicas** per revision

---

*Bài tiếp theo: Bài 5 — Choose and apply revision modes*
