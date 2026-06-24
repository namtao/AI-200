# Bài 3 — Apply KEDA Scalers for Custom Workloads

> Khoá: AI-200 · ACA — Scale containers in Azure Container Apps

---

## Tổng quan

Bài trước bao gồm Azure-native scalers. Bài này mở rộng sang **custom scalers** cho non-Azure event sources: Kafka, Redis, Prometheus metrics, và cron-based scheduling.

---

## Apache Kafka Scaling

Scale dựa trên **consumer group lag** — difference giữa latest offset và committed offset của consumer group.

```yaml
scale:
  minReplicas: 1
  maxReplicas: 50
  rules:
    - name: kafka-scaling
      custom:
        type: kafka
        metadata:
          bootstrapServers: "kafka-broker:9092"
          consumerGroup: "order-consumers"
          topic: "orders"
          lagThreshold: "100"    # lag per partition trigger
        auth:
          - secretRef: kafka-credentials
            triggerParameter: sasl
```

**Công thức:** desired replicas = total lag ÷ lagThreshold

Ví dụ: 500 messages lag, threshold = 100 → 5 replicas cần.

**Giống Event Hubs:** Max effective replicas = số partition. Replica thừa không có partition để đọc.

**Authentication:** SASL/PLAIN, SASL/SCRAM, hoặc TLS — lưu credentials trong Container Apps secrets.

---

## Redis Scaling

Hai loại Redis scaler:

### Redis Lists — Simple queue

Monitor độ dài list bằng `LLEN`. Phù hợp producer-consumer đơn giản.

Key parameters: `address`, `listName`, `listLength` (threshold)

### Redis Streams — Advanced

Monitor **pending entries** trong consumer group — bao gồm cả message đã deliver nhưng chưa acknowledge. Xử lý failure scenario tốt hơn vì tính cả work-in-progress.

**Chọn Redis Lists:** Simple push/pop queue.
**Chọn Redis Streams:** Cần track in-progress work, reliability cao hơn.

ACA hỗ trợ: standard Redis, Redis Cluster, Redis Sentinel — mỗi loại có connection string format khác nhau.

---

## Cron-based Scaling

Scale theo **lịch cố định** thay vì metrics — pre-scale trước giờ cao điểm.

```yaml
scale:
  minReplicas: 0
  maxReplicas: 20
  rules:
    - name: business-hours
      custom:
        type: cron
        metadata:
          timezone: "America/New_York"
          start: "0 8 * * 1-5"     # 8:00 AM, Mon-Fri
          end: "0 18 * * 1-5"      # 6:00 PM, Mon-Fri
          desiredReplicas: "5"      # duy trì 5 replica trong window
    - name: http-scaling
      http:
        metadata:
          concurrentRequests: "50"
```

**Cách hoạt động:**
- Trong time window (8AM-6PM, Mon-Fri): cron scaler request 5 replicas
- Ngoài time window: cron inactive, HTTP scaling tự quyết định
- Nếu không có traffic ngoài giờ → scale về 0

**Khi nhiều rule active:** ACA dùng **highest replica count** giữa tất cả rule.

Ví dụ: cron yêu cầu 5, HTTP yêu cầu 8 → scale up đến 8.

**Use case:** Predictable traffic pattern như business hours, batch job lịch cố định, warm up capacity trước flash sale.

---

## Prometheus Metrics Scaling

Scale dựa trên **custom metrics** từ Prometheus — bất kỳ metric nào app expose.

```yaml
rules:
  - name: prometheus-scaling
    custom:
      type: prometheus
      metadata:
        serverAddress: "http://prometheus:9090"
        metricName: "active_sessions"
        query: "sum(active_user_sessions)"
        threshold: "100"    # desired replicas = query result ÷ threshold
```

`query` nhận **PromQL expression** trả về numeric value.

**Dùng khi:** Standard metrics (HTTP request, queue depth) không phản ánh đúng workload. Ví dụ: scale theo active user sessions, pending transactions, custom business indicator.

---

## Convert KEDA Specification sang Container Apps

Khi migrate từ Kubernetes + KEDA hoặc adapt từ KEDA documentation:

| KEDA field | Container Apps equivalent |
|---|---|
| `triggers[].type` | `--scale-rule-type` hoặc `type` trong YAML |
| `triggers[].metadata` | `--scale-rule-metadata` hoặc `metadata` trong YAML |
| `TriggerAuthentication` | `--scale-rule-auth` + secrets, hoặc `--scale-rule-identity` |
| `minReplicaCount` | `--min-replicas` |
| `maxReplicaCount` | `--max-replicas` |

**Điểm khác biệt:** Container Apps không hỗ trợ full `TriggerAuthentication` resource type — thay vào đó reference secrets trực tiếp trong scale rule. Một số KEDA features nâng cao (external scalers, custom intervals) có thể không available.

---

## Best Practices

**Ưu tiên Azure-native scalers trước:** Service Bus, Event Hubs, Storage Queue có Microsoft support, ít rủi ro hơn community-maintained.

**Test trong staging trước:** Custom scaler có thể có unexpected polling behavior hoặc threshold calculation. Validate pattern scaling trong non-production trước.

**Kết hợp cron + reactive:** Cron để pre-scale trước peak, event-driven/HTTP để handle variation và unexpected spike.

**Document threshold reasoning:** Metadata không self-documenting. Ghi lại tại sao chọn threshold đó, cách authenticate, và metric nào drive scaling.

---

## Bản chất bài này là gì?

**Một câu:** Bài này mở rộng event-driven scaling sang non-Azure sources — nếu bài 2 là "Azure service queue depth", bài này là "bất kỳ queue/metric/schedule nào khác."

**Cron scaling là điểm độc đáo nhất:** Không phải reactive (scale khi có event), mà là **predictive** (scale trước khi event xảy ra). Kết hợp cron (baseline warmup) + HTTP/event (handle variation) = best of both worlds.

**Khi nhiều rule active → highest replica count thắng** — không phải OR, không phải average. Rule nào yêu cầu nhiều replica nhất, đó là số replica được set.

---

## Checklist ghi nhớ cho AI-200

- [ ] **Kafka scaling** dựa trên consumer group lag, max replicas = số partition
- [ ] **Redis Lists** → simple LLEN monitoring · **Redis Streams** → track pending + in-progress
- [ ] **Cron scaling** dùng timezone + cron expression (start/end) + desiredReplicas
- [ ] Cron + HTTP kết hợp: cron đảm bảo baseline, HTTP handle variation
- [ ] Khi nhiều rule active → ACA dùng **highest replica count**
- [ ] **Prometheus scaling** dùng PromQL query, threshold = query result ÷ desired replicas
- [ ] Convert KEDA spec: `triggers[].type` → `--scale-rule-type`, `triggers[].metadata` → `--scale-rule-metadata`
- [ ] Container Apps không support full KEDA `TriggerAuthentication` — dùng secrets hoặc managed identity

---

*Bài tiếp theo: Bài 4 — Select compute resources for performance and cost*
