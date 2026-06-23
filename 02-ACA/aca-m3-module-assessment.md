# Module Assessment — Scale Containers in Azure Container Apps

> Khoá: AI-200 · ACA — Scale containers in Azure Container Apps

---

## Câu 1

**A developer wants to configure a container app that processes messages from an Azure Service Bus queue and scales to zero replicas when no messages are present. Which scaling configuration meets this requirement?**

- Set `--min-replicas 0` with an HTTP scale rule
- Set `--min-replicas 0` with a CPU scale rule ❌
- Set `--min-replicas 0` with an azure-servicebus scale rule ✅

**Đáp án: `--min-replicas 0` với azure-servicebus scale rule**

```bash
az containerapp create \
  --name order-processor \
  --resource-group rg-ecommerce \
  --environment my-environment \
  --image myregistry.azurecr.io/order-processor:v1 \
  --min-replicas 0 \
  --max-replicas 30 \
  --secrets "sb-connection=<CONNECTION_STRING>" \
  --scale-rule-name servicebus-scaling \
  --scale-rule-type azure-servicebus \
  --scale-rule-metadata "queueName=orders" "namespace=sb-ecommerce" "messageCount=5" \
  --scale-rule-auth "connection=sb-connection"
```

Đây là kết hợp đúng: `--min-replicas 0` cho phép scale-to-zero, `azure-servicebus` scaler trigger scale-up khi có message. Khi queue rỗng → không có trigger → scale về 0.

Tại sao các đáp án còn lại sai:
- **HTTP scale rule + min 0:** Message queue processor không nhận HTTP request — không có HTTP traffic để trigger scale-up. App sẽ scale về 0 và không bao giờ tự start vì không có HTTP trigger.
- **CPU scale rule + min 0:** CPU scaling **không thể scale về 0** — cần ít nhất 1 replica đang chạy để đo CPU utilization. Đây là giới hạn cơ bản của resource-based scaling.

---

## Câu 2

**A development team wants to ensure their container app has five replicas ready before the morning traffic spike at 8 AM, while still allowing scale-to-zero overnight. Which scaling approach meets this requirement?**

- Configure a cron scale rule combined with an HTTP scale rule ✅
- Set minimum replicas to five with an HTTP scale rule
- Configure a CPU scale rule with a low utilization threshold

**Đáp án: Cron scale rule kết hợp HTTP scale rule**

```yaml
scale:
  minReplicas: 0    # Scale-to-zero overnight
  maxReplicas: 20
  rules:
    - name: business-hours
      custom:
        type: cron
        metadata:
          timezone: "America/New_York"
          start: "0 8 * * 1-5"     # 8 AM weekdays
          end: "0 18 * * 1-5"
          desiredReplicas: "5"      # Pre-scale 5 replicas
    - name: http-scaling
      http:
        metadata:
          concurrentRequests: "50"  # Handle traffic variation
```

Cron rule đảm bảo 5 replica có sẵn từ 8 AM. `minReplicas: 0` cho phép scale-to-zero ngoài giờ. HTTP rule xử lý variation trong giờ làm việc.

Tại sao các đáp án còn lại sai:
- **`minReplicas: 5` + HTTP:** `minReplicas: 5` duy trì 5 replica **24/7** — không scale-to-zero được overnight. Câu hỏi yêu cầu cả hai: pre-scale 8 AM và scale-to-zero overnight.
- **CPU scale rule + low threshold:** CPU scale không thể scale-to-zero, và không có cơ chế pre-scale theo lịch. CPU thấp overnight → scale down, nhưng không về 0 và không đảm bảo 5 replica lúc 8 AM.

---

## Câu 3

**A team is deploying a new version of their container app and wants to send 10% of traffic to the new version for validation before full rollout. What must they configure?**

- Enable multiple revision mode and configure traffic splitting with revision weights ✅
- Use single revision mode with zero-downtime deployment
- Configure a cron scale rule to schedule traffic shifts

**Đáp án: Multiple revision mode + traffic splitting**

```bash
# Bước 1: Enable multiple revision mode
az containerapp update \
  --name order-api \
  --resource-group rg-ecommerce \
  --revision-mode multiple

# Bước 2: Deploy version mới → tạo revision v2

# Bước 3: Set traffic splitting
az containerapp ingress traffic set \
  --name order-api \
  --resource-group rg-ecommerce \
  --revision-weight order-api--v1=90 order-api--v2=10
```

Traffic splitting chỉ available trong **multiple revision mode**. 10% traffic đến v2 để validate trước khi shift hoàn toàn.

Tại sao các đáp án còn lại sai:
- **Single revision mode + zero-downtime:** Single mode tự động shift **100% traffic** sang revision mới sau khi healthy. Không có cơ chế chỉ send 10%. Zero-downtime deployment trong single mode là all-or-nothing.
- **Cron scale rule:** Cron rule kiểm soát **số lượng replica**, không kiểm soát **traffic distribution**. Không có cách nào dùng cron để schedule "chuyển 10% traffic lúc này, 50% lúc kia".

---

## Câu 4

**When configuring KEDA scale rules for Azure services, which authentication method is recommended for production workloads?**

- Connection strings stored as container app secrets
- Managed identity with `--scale-rule-identity` ✅
- Anonymous access without authentication

**Đáp án: Managed identity với `--scale-rule-identity`**

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
  --scale-rule-metadata "accountName=stecommerce" "queueName=inventory-updates" "queueLength=10" \
  --scale-rule-identity <MANAGED_IDENTITY_RESOURCE_ID>
```

Managed identity là recommended vì:
- **Không có long-lived credential** cần lưu hay rotate
- **Không risk credential exposure** trong script hay config
- **Align với Azure security best practices** cho service-to-service auth
- Assign RBAC role tối thiểu cần thiết (vd. Azure Service Bus Data Receiver)

Tại sao các đáp án còn lại sai:
- **Connection strings as secrets:** Works nhưng kém hơn managed identity cho production — phải lưu credential, phải rotate thủ công, risk exposure nếu secret bị leak. Đây là acceptable cho development nhưng không recommended cho production.
- **Anonymous access:** Không phải option thực tế cho Azure services — Service Bus, Storage Queue đều yêu cầu authentication. Anonymous access không tồn tại cho các service này.

---

## Tổng kết — Kết quả 4/4

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Scale-to-zero cho queue processor | `--min-replicas 0` + azure-servicebus rule |
| 2 | Pre-scale theo lịch + scale-to-zero overnight | Cron rule + HTTP rule kết hợp |
| 3 | 10% canary traffic | Multiple revision mode + traffic splitting |
| 4 | Authentication cho KEDA scalers production | Managed identity (`--scale-rule-identity`) |

---

*Module hoàn thành. Learning Path ACA — Scale containers in Azure Container Apps kết thúc.*
