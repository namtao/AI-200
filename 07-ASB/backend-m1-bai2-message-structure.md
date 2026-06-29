# Bài 2 — Structure Messages for AI Workloads

> Khoá: AI-200 · Service Bus — Queue and process AI operations with Azure Service Bus

---

## Message Anatomy

| Part | Nội dung |
|---|---|
| **Body** | Primary data payload (thường là JSON) |
| **Application properties** | Custom key-value metadata (routing, tracking) |
| **System properties** | Set bởi Service Bus/SDK: `message_id`, `correlation_id`, `content_type`, `time_to_live`, `session_id`, `sequence_number` |

---

## Tạo Message với JSON Payload

```python
import json
import uuid
from azure.servicebus import ServiceBusMessage

request_payload = {
    "prompt": "Extract the contract parties and effective date.",
    "model": "gpt-4o",
    "temperature": 0.1,
    "max_tokens": 2000,
    "document_id": "doc-2025-0142"
}

message = ServiceBusMessage(
    body=json.dumps(request_payload),
    content_type="application/json",
    message_id=str(uuid.uuid4()),      # Unique ID per message
    correlation_id="req-abc-12345",    # End-to-end tracking
    application_properties={
        "model_name": "gpt-4o",
        "priority": "standard",
        "document_type": "contract"
    }
)
```

`message_id` — unique identifier. Enable duplicate detection trên queue → Service Bus discard duplicate submissions trong detection window.

---

## Correlation ID — End-to-end Tracking

`correlation_id` follow request qua toàn bộ pipeline:

```
API enqueue → Processor run inference → Downstream services → Results store
      ↑                  ↑                     ↑                  ↑
 cùng correlation_id trên tất cả stages
```

Troubleshoot failed inference: search logs, DLQ, result stores theo correlation_id → trace full lifecycle.

**Với OpenTelemetry:** Propagate trace context qua application properties:

```python
from opentelemetry import trace

current_span = trace.get_current_span()
span_context = current_span.get_span_context()

message = ServiceBusMessage(
    body=json.dumps(request_payload),
    correlation_id=correlation_id,
    application_properties={
        "traceparent": f"00-{format(span_context.trace_id, '032x')}-{format(span_context.span_id, '016x')}-01",
        "model_name": "gpt-4o"
    }
)
```

---

## Claim-Check Pattern — Large Payloads

Service Bus Standard: max **256 KB**. Premium: max **100 MB**. AI workloads có thể vượt limit (full documents, images, batch vectors).

**Giải pháp:** Upload large payload lên Blob Storage, message chứa blob URI.

```python
from azure.storage.blob import BlobServiceClient

# Upload large payload lên Blob
blob_service = BlobServiceClient(account_url="https://<storage>.blob.core.windows.net", credential=credential)
blob_client = blob_service.get_container_client("documents").get_blob_client("doc-2025-0142.pdf")
blob_client.upload_blob(large_document_bytes)

# Send claim-check message (nhỏ)
claim_check = {
    "blob_uri": "https://<storage>.blob.core.windows.net/documents/doc-2025-0142.pdf",
    "document_id": "doc-2025-0142",
    "model": "gpt-4o",
    "operation": "extract"
}

message = ServiceBusMessage(
    body=json.dumps(claim_check),
    content_type="application/json",
    correlation_id="req-abc-12345",
    application_properties={"pattern": "claim-check"}
)
```

Consumer retrieve full payload từ blob URI, process, optionally delete blob.

**Lợi ích:**
- Work within any tier's size limits
- Giảm broker throughput costs (chỉ transfer small reference messages)
- Separate access controls cho message metadata vs payload

---

## Time-to-Live (TTL)

```python
from datetime import timedelta

# Real-time inference — expire sau 5 phút
urgent_message = ServiceBusMessage(
    body=json.dumps(request_payload),
    time_to_live=timedelta(minutes=5)
)

# Batch processing — expire sau 24 giờ
batch_message = ServiceBusMessage(
    body=json.dumps(batch_payload),
    time_to_live=timedelta(hours=24)
)
```

Expired messages → dead-letter queue (nếu "dead-lettering on expiration" enabled) → visibility into unprocessed messages.

---

## Batch Sending

```python
with client.get_queue_sender("inference-requests") as sender:
    batch = sender.create_message_batch()

    for request in pending_requests:
        message = ServiceBusMessage(body=json.dumps(request), content_type="application/json")
        try:
            batch.add_message(message)
        except MessageSizeExceededError:
            sender.send_messages(batch)     # Send full batch
            batch = sender.create_message_batch()
            batch.add_message(message)      # Start new batch

    if len(batch) > 0:
        sender.send_messages(batch)         # Send remaining
```

SDK manage batch size automatically. `MessageSizeExceededError` → send current batch, start new.

---

## Bản chất bài này là gì?

**Một câu:** Message structure trong Service Bus gồm 3 lớp (body, application properties, system properties) — biết dùng đúng lớp cho đúng mục đích: correlation tracking, large payload, TTL, và batch sending.

### So sánh với Kafka / RabbitMQ / AWS SQS

| | Kafka | RabbitMQ | AWS SQS | Service Bus |
|---|---|---|---|---|
| Max message size | 1 MB (default) | 128 MB | 256 KB | Standard: 256 KB · Premium: 100 MB |
| Large payload | Không native | Không native | Không native | Claim-check pattern (Blob) |
| Metadata | Headers | Headers/Properties | Message attributes | Application properties |
| Duplicate detection | Consumer side | Consumer side | Không built-in | `message_id` + detection window |
| TTL | Retention policy (topic) | Per-message TTL | Visibility timeout | Per-message `time_to_live` |
| Batch send | Producer batching | Publisher confirms | Batch up to 10 | `create_message_batch()` |

**Claim-check pattern là gì:** Upload large file lên Blob Storage, chỉ gửi URI qua Service Bus. Consumer download từ Blob. Tránh vượt message size limit mà không cần tăng tier.

**Exam trap:** `message_id` = duplicate detection. `correlation_id` = end-to-end tracking. Hai thứ này không thể hoán đổi — nhầm = sai câu assessment.

---

## Checklist ghi nhớ cho AI-200

- [ ] Message anatomy: **body** (payload) · **application properties** (custom metadata) · **system properties** (Service Bus managed)
- [ ] `content_type = "application/json"` signal encoding format
- [ ] `message_id` = unique per message · enable duplicate detection = discard duplicates in window
- [ ] `correlation_id` = end-to-end tracking qua toàn pipeline
- [ ] Standard tier: max **256 KB** · Premium tier: max **100 MB**
- [ ] **Claim-check pattern:** Upload lên Blob → send URI in message
- [ ] `time_to_live` = message expiration (real-time: minutes · batch: hours)
- [ ] Expired messages → DLQ nếu dead-lettering on expiration enabled
- [ ] Batch: `create_message_batch()` → `add_message()` → handle `MessageSizeExceededError`

---

*Bài tiếp theo: Bài 3 — Process messages reliably*
