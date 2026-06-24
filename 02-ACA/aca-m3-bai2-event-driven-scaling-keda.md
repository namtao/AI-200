# Bài 2 — Implement Event-Driven Scaling with KEDA

> Khoá: AI-200 · ACA — Scale containers in Azure Container Apps

---

## Tại sao cần event-driven scaling?

HTTP scaling tốt cho synchronous API. Nhưng nhiều AI workload là **asynchronous** — xử lý message từ queue, consume event stream, chạy batch job. Với các workload này, HTTP request volume không phản ánh actual work cần làm.

Ví dụ: Queue có 500 message chờ xử lý nhưng không có HTTP request nào đến → HTTP scaling không trigger → không có replica nào xử lý queue.

Event-driven scaling giải quyết bằng cách scale dựa trên **external signals**: queue depth, event stream lag, custom metrics.

---

## KEDA Integration

ACA dùng KEDA để extend scaling beyond HTTP/TCP. KEDA monitor external event source và điều chỉnh replica count dựa trên metrics như queue depth, partition lag.

**Polling interval:** 30 giây — platform check event source mỗi 30s.

**Scaler categories:**
- **Microsoft-maintained (Azure-native):** Service Bus, Event Hubs, Storage Queue, Blob Storage, Log Analytics, Azure Monitor — recommended, first-party support
- **Community-maintained:** Kafka, Redis, Prometheus, Cron, PostgreSQL, MySQL... — quality varies
- **External scalers:** Cần deploy riêng, không covered trong module

Event-driven scaling **hỗ trợ scale-to-zero** — ideal cho background processor.

---

## Azure Service Bus Scaling

Scale dựa trên **message count trong queue hoặc topic subscription**.

**Công thức:** desired replicas = current message count ÷ messageCount threshold

Ví dụ: 50 messages, threshold = 5 → cần 10 replicas.

```bash
az containerapp create \
  --name order-processor \
  --resource-group rg-ecommerce \
  --environment my-environment \
  --image myregistry.azurecr.io/order-processor:v1 \
  --min-replicas 0 \
  --max-replicas 30 \
  --secrets "sb-connection=<SERVICE_BUS_CONNECTION_STRING>" \
  --scale-rule-name servicebus-scaling \
  --scale-rule-type azure-servicebus \
  --scale-rule-metadata "queueName=orders" \
                        "namespace=sb-ecommerce" \
                        "messageCount=5" \
  --scale-rule-auth "connection=sb-connection"
```

Key parameters:
- `queueName` hoặc `topicName` + `subscriptionName` (cho topic)
- `namespace` — Service Bus namespace
- `messageCount` — messages per replica threshold
- `--scale-rule-auth` — map secret name đến auth parameter của scaler

**Topic subscription:** Thay `queueName` bằng `topicName` và thêm `subscriptionName`. Mỗi subscription có message count riêng → consumer apps scale độc lập.

---

## Azure Storage Queue Scaling

Alternative đơn giản hơn Service Bus, chi phí thấp hơn.

Key parameters: `queueName`, `accountName`, `queueLength` (messages per replica)

**Chọn Storage Queue khi:** Simple producer-consumer, không cần sessions/dead-letter/scheduled messages.
**Chọn Service Bus khi:** Cần advanced features, higher throughput, guaranteed ordering.

---

## Azure Event Hubs Scaling

Scale dựa trên **partition lag** — số event chưa được xử lý trong consumer group.

```yaml
# YAML fragment
scale:
  minReplicas: 0
  maxReplicas: 32
  rules:
    - name: eventhubs-scaling
      custom:
        type: azure-eventhub
        metadata:
          consumerGroup: "$Default"
          unprocessedEventThreshold: "64"    # events per partition trigger
          checkpointStrategy: "blobMetadata"
        auth:
          - secretRef: eh-connection
            triggerParameter: connection
```

**Giới hạn quan trọng:** Số replica không thể hiệu quả hơn số partition. 32 partition → max 32 replica có ích. Replica thứ 33 trở đi không có partition để đọc.

**`checkpointStrategy: blobMetadata`** — recommended khi dùng Azure Blob Storage cho checkpointing.

---

## Authenticate Scale Rules

Hai cách xác thực với event source:

### Secrets-based (đơn giản hơn)

```bash
--secrets "sb-connection=<CONNECTION_STRING>"
--scale-rule-auth "connection=sb-connection"
```

Nhược điểm: Phải lưu connection string, phải rotate thủ công.

### Managed Identity (recommended cho production)

```bash
az containerapp create \
  --name queue-processor \
  --resource-group rg-ecommerce \
  --environment my-environment \
  --image myregistry.azurecr.io/queue-processor:v1 \
  --user-assigned <MANAGED_IDENTITY_RESOURCE_ID> \
  --min-replicas 0 \
  --max-replicas 20 \
  --scale-rule-name storage-queue-scaling \
  --scale-rule-type azure-queue \
  --scale-rule-metadata "accountName=stecommerce" \
                        "queueName=inventory-updates" \
                        "queueLength=10" \
  --scale-rule-identity <MANAGED_IDENTITY_RESOURCE_ID>
```

`--scale-rule-identity` — chỉ định managed identity để scaler dùng khi authenticate.
Cần gán đúng RBAC role (vd. **Azure Service Bus Data Receiver** cho Service Bus).

---

## Setting messageCount Threshold

Tính threshold dựa trên processing time:

```
Mỗi message xử lý 10 giây
Muốn 100 messages/phút = 100/60 ≈ 2 messages/giây
Cần: 2 msg/s × 10s/msg = 20 concurrent workers
→ messageCount = 100 messages / 20 replicas = 5
```

---

## Bản chất bài này là gì?

**Một câu:** Event-driven scaling giải quyết bài toán HTTP scaling không giải quyết được — scale dựa trên "work waiting to be done" thay vì "requests happening right now."

### So sánh với App Service và Docker

| | Docker | App Service | ACA Event-driven |
|---|---|---|---|
| Scale theo queue depth | ❌ | ❌ | ✅ Service Bus, Storage Queue |
| Scale theo event lag | ❌ | ❌ | ✅ Event Hubs, Kafka |
| Scale-to-zero khi queue rỗng | ❌ | ❌ | ✅ |
| Poll interval | N/A | N/A | 30 giây |

**Mental model:** HTTP scale = "có bao nhiêu người đang dùng ngay bây giờ?". Event-driven scale = "có bao nhiêu việc cần làm trong queue?". Hai câu hỏi khác nhau, hai workload pattern khác nhau.

**Công thức chung:** `desired replicas = queue_depth ÷ messages_per_replica`. Với Event Hubs: max effective replicas = số partition (replica thừa không có partition để read).

---

## Checklist ghi nhớ cho AI-200

- [ ] Event-driven scaling dùng KEDA, polling mỗi **30 giây**
- [ ] **Azure-native scalers** (Service Bus, Event Hubs, Storage Queue...) có Microsoft first-party support
- [ ] Service Bus: scale theo message count, formula = messages ÷ threshold
- [ ] Topic subscription: mỗi subscription scale độc lập theo message count riêng
- [ ] Storage Queue: đơn giản hơn Service Bus, chi phí thấp hơn, ít tính năng hơn
- [ ] Event Hubs: scale theo **partition lag**, max effective replicas = số partition
- [ ] `checkpointStrategy: blobMetadata` recommended cho Event Hubs
- [ ] Auth: **managed identity** (`--scale-rule-identity`) recommended cho production
- [ ] Managed identity cần RBAC role phù hợp trên resource (vd. Service Bus Data Receiver)
- [ ] Event-driven scaling hỗ trợ **scale-to-zero** — ideal cho background processor

---

*Bài tiếp theo: Bài 3 — Apply KEDA scalers for custom workloads*
