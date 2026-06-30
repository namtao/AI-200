# Bài 1 — Configure Scale Rules

> Khoá: AI-200 · ACA — Scale containers in Azure Container Apps

---

## Tổng quan

ACA dùng **declarative scale rules** — bạn khai báo điều kiện trigger, platform tự thêm/bớt replica. Bài này bao gồm 4 loại scale rule cơ bản: HTTP, TCP, CPU, Memory.

ACA dùng **KEDA (Kubernetes Event-driven Autoscaling)** làm nền tảng scaling — platform dịch config của bạn thành KEDA specification.

---

## Scale Definitions — 3 thành phần

```
Scale definition
├── Limits   → min/max replica count
├── Rules    → trigger conditions (HTTP, TCP, CPU, memory, event-driven)
└── Behavior → timing parameters (polling interval, cool-down, scale-up steps)
```

### Default behavior

- Default: 0–10 replica khi ingress enabled, không có custom rule
- Nếu ingress disabled và không có min replica/custom rule → scale-to-zero và **không bao giờ restart** vì không có trigger

### Billing

| State | Chi phí |
|---|---|
| Zero replicas | Không tính compute |
| Running nhưng không xử lý request | Idle rate (thấp hơn active) |
| Running và xử lý request | Active rate |

---

## HTTP Scale Rules

Scale dựa trên **concurrent HTTP requests**. Platform tính trung bình request trong 15 giây qua.

Default threshold: **10 concurrent requests/replica**.

```bash
az containerapp create \
  --name order-api \
  --resource-group rg-ecommerce \
  --environment my-environment \
  --image myregistry.azurecr.io/order-api:v1 \
  --min-replicas 1 \
  --max-replicas 10 \
  --scale-rule-name http-scaling \
  --scale-rule-type http \
  --scale-rule-http-concurrency 50
```

`--scale-rule-http-concurrency 50` = khi một replica nhận hơn 50 concurrent request → scale up thêm replica.

**Khi nào dùng HTTP scaling:**
- Synchronous API workload
- Web application
- Request volume correlate trực tiếp với resource needs
- Cần scale-to-zero

**Lưu ý:** Container Apps Jobs không dùng được HTTP scaling vì không expose HTTP endpoint.

### Threshold thấp vs cao

| Threshold | Effect |
|---|---|
| Thấp (vd. 5) | Scale sớm hơn, nhiều replica hơn, headroom tốt hơn |
| Cao (vd. 100) | Maximize utilization mỗi replica, nhưng latency tăng trước khi replica mới ready |

---

## TCP Scale Rules

Scale dựa trên **concurrent TCP connections** — cho persistent connection thay vì request-response.

```bash
az containerapp create \
  --name websocket-server \
  --resource-group rg-demo \
  --environment my-environment \
  --image myregistry.azurecr.io/ws-server:v1 \
  --min-replicas 1 \
  --max-replicas 20 \
  --scale-rule-name tcp-scaling \
  --scale-rule-type tcp \
  --scale-rule-tcp-concurrency 100
```

**Dùng TCP scaling cho:**
- WebSocket server
- gRPC service
- Database connection pool
- Bất kỳ app nào dùng long-lived connection

TCP cũng support scale-to-zero: khi tất cả connection đóng, sau cool-down period ACA scale về 0.

---

## CPU và Memory Scale Rules

Scale dựa trên **resource utilization** (percentage).

```yaml
# YAML — kết hợp HTTP và CPU
scale:
  minReplicas: 1
  maxReplicas: 20
  rules:
    - name: http-scaling
      http:
        metadata:
          concurrentRequests: "100"
    - name: cpu-scaling
      custom:
        type: cpu
        metadata:
          type: Utilization
          value: "70"       # Scale khi CPU > 70%
```

**Hạn chế quan trọng:** CPU và Memory scale rules **KHÔNG thể scale về 0**. Platform cần ít nhất 1 replica đang chạy để đo utilization.

> Nếu cần scale-to-zero: dùng HTTP hoặc event-driven rule làm primary trigger, kết hợp thêm CPU/memory.

**Dùng CPU scaling:** Compute-intensive workload — image processing, ML inference, video transcoding.
**Dùng Memory scaling:** Memory-intensive — caching, data aggregation, large payload processing.

---

## Scale Behavior — Timing Parameters

| Parameter | Giá trị | Ý nghĩa |
|---|---|---|
| Polling interval (custom/CPU/memory) | 30 giây | Bao lâu check trigger một lần |
| HTTP/TCP calculation window | 15 giây | Window tính average request/connection |
| Scale-up stabilization | 0 giây | Scale up **ngay lập tức** khi vượt threshold |
| Scale-down stabilization | 300 giây (5 phút) | Chờ 5 phút trước khi scale down |
| Cool-down period (to zero) | 300 giây | Chờ 5 phút sau lần traffic cuối trước khi về 0 |

**Scale-up pattern:** 1 → 4 → 8 → 16 → 32... (double mỗi step) cho đến max.

**Scale-down:** Xoá tất cả replica cần giảm **cùng lúc**, không dần dần.

### Ý nghĩa thực tế của cool-down 300 giây

Workload bursty (traffic ngắt quãng) → replicas tiếp tục chạy 5 phút sau khi traffic dừng → tốn chi phí idle. Cần tính đến điều này khi tính toán cost optimization.

---

## Kết hợp nhiều scale rules

Khi có nhiều rule, ACA scale out khi **bất kỳ rule nào** trigger:

```yaml
# Scale out khi HTTP > 100 req/replica HOẶC CPU > 70%
rules:
  - name: http-scaling
    http:
      metadata:
        concurrentRequests: "100"
  - name: cpu-scaling
    custom:
      type: cpu
      metadata:
        type: Utilization
        value: "70"
```

---

## Bản chất bài này là gì?

**Một câu:** ACA autoscaling là KEDA wrapped trong một API đơn giản hơn — bạn khai báo trigger condition, platform tự scale; App Service không có gì tương đương phía KEDA.

### So sánh với App Service và Kubernetes

| Scale trigger | App Service | Kubernetes HPA | ACA |
|---|---|---|---|
| HTTP request | Manual hoặc autoscale rules | ❌ không sẵn | ✅ HTTP scale rule |
| CPU/Memory | ✅ autoscale | ✅ HPA | ✅ nhưng không scale-to-zero |
| Queue depth (Service Bus, Storage) | ❌ | ✅ qua KEDA | ✅ event-driven |
| Custom metrics | ❌ | ✅ custom metrics | ✅ Prometheus |
| Scale-to-zero | ❌ (chỉ Free/Shared) | ✅ qua KEDA | ✅ HTTP, event-driven |

**Điểm quan trọng nhất:** CPU/Memory scale **không thể scale về 0** — cần ít nhất 1 replica để đo. Chỉ HTTP và event-driven scale mới hỗ trợ scale-to-zero.

**Scale-up ngay, scale-down chờ 5 phút** — pattern này quan trọng khi tính chi phí workload bursty.

---

## Checklist ghi nhớ cho AI-200

- [ ] Scale definitions gồm 3 phần: **limits**, **rules**, **behavior**
- [ ] ACA dùng **KEDA** làm underlying scaling infrastructure
- [ ] Default: 0–10 replica khi ingress enabled
- [ ] **HTTP scaling** → concurrent requests/15s window, hỗ trợ scale-to-zero
- [ ] **TCP scaling** → concurrent connections, dùng cho WebSocket/gRPC/persistent connection
- [ ] **CPU/Memory scaling** → **KHÔNG scale về 0** — cần ít nhất 1 replica để đo
- [ ] Default HTTP threshold: **10 concurrent requests/replica**
- [ ] Polling interval: HTTP/TCP = **15s** · Custom/CPU/Memory = **30s**
- [ ] Scale-up: ngay lập tức (0s stabilization), pattern 1→4→8→16...
- [ ] Scale-down: chờ **300 giây** (5 phút) trước khi reduce
- [ ] Khi nhiều rule: scale out khi **bất kỳ rule nào** trigger

---

[← Assessment](./aca-m2-module-assessment.md) · [🏠 Mục lục](../README.md) · [Bài 2 →](./aca-m3-bai2-event-driven-scaling-keda.md)
