# Bài 3 — Configure Delivery and Retry Policies for Reliable Event Processing

> Khoá: AI-200 · Event Grid — Develop event-driven AI workflows with Azure Event Grid

---

## Delivery Mechanics

Event Grid gửi HTTP POST đến subscriber endpoint. Subscriber phải respond success code **(200, 201, 202, 203, 204)** để acknowledge. Timeout: **30 giây**. Bất kỳ error code khác hoặc timeout → retry.

**At-least-once delivery:** Handler có thể nhận cùng event nhiều hơn một lần → handler phải **idempotent**. Dùng `id` field để detect và deduplicate.

---

## Không Retry (Permanent errors)

| Code | Lý do |
|---|---|
| **400 Bad Request** | Malformed payload |
| **413 Request Entity Too Large** | Exceeds size limit |
| **403 Forbidden** | Endpoint rejects explicitly |
| 401 Unauthorized (webhooks) | Not retried |

---

## Retry Schedule (Exponential backoff)

| Attempt | Wait |
|---|---|
| 1 | 10 giây |
| 2 | 30 giây |
| 3 | 1 phút |
| 4 | 5 phút |
| 5 | 10 phút |
| 6 | 30 phút |
| 7 | 1 giờ |
| 8 | 3 giờ |
| 9 | 6 giờ |
| 10+ | Mỗi 12 giờ, đến 24 giờ |

Event Grid thêm randomization để spread load.

---

## Customize Retry Policy

2 parameters — Event Grid dùng cái nào **đến trước** để stop retry:

```bash
az eventgrid event-subscription create \
    --name moderation-sub \
    --source-resource-id /subscriptions/{sub-id}/.../topics/ai-events \
    --endpoint https://moderation-service.azurewebsites.net/api/events \
    --max-delivery-attempts 5 \   # Default 30, range 1-30
    --event-ttl 30                # Default 1440 min (24h), range 1-1440
```

**Time-sensitive AI (real-time classification):** TTL thấp (30 min) → không waste resources trên stale events.
**Batch processing:** TTL cao (24h), nhiều attempts.

**Lưu ý:** Với TTL=30 phút và exponential backoff, chỉ khoảng 6 delivery attempts complete — setting max 10 với TTL 30 không có effect thêm.

---

## Dead-Letter Destination

Sau khi exhaust all retries hoặc TTL expire → event vào dead-letter. **Disabled by default.** Cần Azure Blob Storage container.

```bash
az eventgrid event-subscription create \
    --name moderation-sub \
    --endpoint https://moderation-service.azurewebsites.net/api/events \
    --max-delivery-attempts 5 \
    --event-ttl 30 \
    --deadletter-endpoint /subscriptions/{sub-id}/.../storageAccounts/{account}/blobServices/default/containers/dead-letters
```

Dead-lettered event properties:
- `deadLetterReason`: `MaxDeliveryAttemptsExceeded` hoặc `MaxRetryDurationExceeded`
- `deliveryAttempts`: Số lần đã try
- `lastDeliveryOutcome`: `NotFound`, `TimedOut`, `Busy`, `Forbidden`
- `publishTime`: Khi Event Grid accept event
- `lastDeliveryAttemptTime`

---

## Transient Failures — Handler Best Practices

```
Return 200-204 → success
Return 503 → temporarily overloaded → retry
Return 400 → NOT retried (permanent error)
Return 202 → accepted, process async (cho operations > 30 giây)
```

**Cold start problem:** Azure Functions consumption plan có thể exceed 30s response timeout. Return **202 Accepted** ngay → process async.

---

## Output Batching

```bash
az eventgrid event-subscription create \
    --name batch-processor-sub \
    --endpoint https://batch-processor.azurewebsites.net/api/events \
    --max-events-per-batch 100 \           # 1-5000
    --preferred-batch-size-in-kilobytes 512  # 1-1024
```

**All-or-none semantics:** Handler phải return success cho toàn bộ batch. Nếu fail → entire batch retried.

---

## Azure Monitor Metrics

| Metric | Dùng để detect |
|---|---|
| Delivery success count | Normal flow |
| **Dead-lettered events** ↑ | Model service down, handler URL changed |
| **Matched events** ↓ | Event source stopped publishing, filter changed |
| Delivery failure count | Individual attempt failures |
| Dropped events | Exceeded retry limits, no dead-letter |

---

## Bản chất bài này là gì?

**Một câu:** Event Grid delivery = at-least-once với exponential backoff retry — handler phải idempotent, trả 202 cho async ops > 30s, và configure dead-letter (disabled by default) để không mất events sau khi exhaust retries.

### So sánh với Service Bus / AWS SNS / Webhook retry

| | Event Grid | Service Bus | AWS SNS | Webhook (tự implement) |
|---|---|---|---|---|
| Retry mechanism | Exponential backoff, tự động | Peek-lock + max delivery count | 3 attempts, then DLQ | Tự implement |
| Max retry window | 24h (event TTL) | Configurable (days) | 23 giờ | Tuỳ |
| Dead-letter | Blob Storage container (opt-in) | Automatic subqueue | SQS (opt-in) | Tuỳ |
| Dead-letter default | **Disabled** | Enabled (on max count) | Disabled | N/A |
| HTTP timeout | 30 giây | N/A | 15 giây | Tuỳ |
| No-retry codes | 400, 413, 403 | N/A | 4xx (varies) | Tuỳ |
| Batching | All-or-none per batch | Per message | N/A | N/A |

**Exponential backoff pattern:** Event Grid không retry ngay — tăng dần từ 10s → 30s → ... → 12h. Với TTL 30 phút, chỉ ~6 attempts thực tế mặc dù set max-delivery-attempts = 30.

**Exam trap:** Dead-letter **disabled by default** trong Event Grid — khác với Service Bus (DLQ luôn tồn tại). Cần explicitly configure `--deadletter-endpoint` trỏ Blob container. Không config = events dropped permanently sau retry exhaustion.

---

## Checklist ghi nhớ cho AI-200

- [ ] Response success: **200, 201, 202, 203, 204** · Timeout: **30 giây**
- [ ] **At-least-once** → handlers phải idempotent, check `id`
- [ ] 400, 413, 403 → **không retry**
- [ ] Exponential backoff: 10s → 30s → 1m → 5m → 10m → 30m → 1h → 3h → 6h → 12h
- [ ] `--max-delivery-attempts` (1-30, default 30) · `--event-ttl` (1-1440 min, default 1440)
- [ ] Event Grid stops khi đến **limit nào trước**
- [ ] Dead-letter: **disabled by default**, cần Blob Storage container
- [ ] Dead-letter reasons: `MaxDeliveryAttemptsExceeded` · `MaxRetryDurationExceeded`
- [ ] Return **202** cho operations > 30s (accept immediately, process async)
- [ ] Batching: all-or-none semantics cho entire batch

---

[← Bài 2](./backend-m2-bai2-schemas.md) · [🏠 Mục lục](../README.md) · [Bài 4 →](./backend-m2-bai4-publish-events.md)
