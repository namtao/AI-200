# Module Assessment — Queue and Process AI Operations with Azure Service Bus

> Khoá: AI-200 · Service Bus — Queue and process AI operations with Azure Service Bus

---

## Câu 1

**Your AI platform completes a document analysis, and three independent services need to react to the result: a notification service alerts the user, an audit service logs the result for compliance, and a dashboard service updates metrics. Which Service Bus entity type supports this requirement?**

- A topic with three subscriptions ✅
- A queue with three competing consumers
- Three separate queues with the sender publishing to each one

**Đáp án: A topic with three subscriptions**

Topic delivers một **copy** của message đến **tất cả** subscriptions. Ba services (notification, audit, dashboard) mỗi cái có subscription riêng → nhận cùng message và process độc lập. Failure ở một subscription không ảnh hưởng các subscription khác.

Publisher chỉ cần publish đến một topic — không cần biết bao nhiêu subscriptions hay chúng làm gì.

Tại sao các đáp án còn lại sai:
- **Queue với 3 competing consumers:** Queue deliver mỗi message đến **đúng một** consumer — 3 workers compete để nhận message. Chỉ một worker xử lý, hai kia không thấy message này. Không đạt requirement "3 services đều phản ứng."
- **3 separate queues:** Works nhưng là antipattern — sender phải explicitly publish đến 3 queues, bị coupled với consumers. Thêm service thứ 4 = sửa publisher code. Topic giải quyết elegant hơn.

---

## Câu 2

**You're building an AI inference pipeline where losing a customer's request is unacceptable. If a worker crashes while processing a message, the message must become available to another worker. Which receive mode should you configure?**

- Peek-lock mode ✅
- Receive-and-delete mode
- Deferred receive mode

**Đáp án: Peek-lock mode**

Peek-lock: Service Bus **lock** message, deliver đến consumer, nhưng **không remove** khỏi queue. Consumer phải explicitly settle (complete/abandon) trong lock duration. Nếu worker crash → lock expire → message available lại cho worker khác.

```python
with client.get_queue_receiver(
    queue_name="inference-requests",
    receive_mode=ServiceBusReceiveMode.PEEK_LOCK,
) as receiver:
    for msg in receiver:
        try:
            run_inference(msg)
            receiver.complete_message(msg)
        except:
            receiver.abandon_message(msg)  # Or let lock expire
```

Tại sao các đáp án còn lại sai:
- **Receive-and-delete:** Remove message **ngay khi deliver** — nếu worker crash sau khi nhận nhưng trước khi complete inference → message **lost forever**. Unacceptable cho critical AI requests.
- **Deferred receive:** Không phải receive mode — đây là settlement operation. `defer_message()` keeps message in queue nhưng removes from delivery, only retrievable by sequence number. Không liên quan đến crash recovery.

---

## Câu 3

**A message in your inference queue consistently causes a processing error every time a worker attempts it. After 10 delivery attempts, what does Service Bus do with the message?**

- Moves the message to the dead-letter queue with the reason MaxDeliveryCountExceeded ✅
- Deletes the message permanently from the queue
- Returns the message to the back of the queue for continued retry attempts

**Đáp án: Move to dead-letter queue với MaxDeliveryCountExceeded**

Max delivery count (default = 10) ngăn poison message block queue indefinitely. Sau 10 delivery attempts, Service Bus tự động move message đến dead-letter queue với reason `MaxDeliveryCountExceeded`.

```python
with client.get_queue_receiver("inference-requests",
                               sub_queue=ServiceBusSubQueue.DEAD_LETTER) as dlq_receiver:
    for msg in dlq_receiver:
        print(f"Reason: {msg.dead_letter_reason}")  # MaxDeliveryCountExceeded
        print(f"Delivery count: {msg.delivery_count}")  # 10
```

Tại sao các đáp án còn lại sai:
- **Deletes permanently:** Service Bus không xóa message mà không có explicit action. DLQ preserves message cho investigation — quan trọng để debug systemic issues.
- **Continues retry indefinitely:** Đây là điều max delivery count ngăn chặn. Infinite retry của poison message = waste processing resources, block queue, mask real failure count.

---

## Câu 4

**Your AI pipeline needs to process 500-MB document files, but Azure Service Bus Premium tier supports messages up to 100 MB via AMQP. Which pattern addresses this constraint?**

- The claim-check pattern, uploading files to Azure Blob Storage and sending only the blob URI in the message ✅
- Splitting the document into five 100-MB messages and reassembling them at the consumer
- Encoding the document as base64 and sending it as the message body on the Premium tier

**Đáp án: Claim-check pattern**

```python
# Upload 500 MB file to Blob
blob_client.upload_blob(large_document_bytes)

# Send small claim-check message với blob URI
claim_check = {
    "blob_uri": "https://<storage>.blob.core.windows.net/documents/doc-500mb.pdf",
    "document_id": "doc-2025-0142",
    "model": "gpt-4o"
}
message = ServiceBusMessage(body=json.dumps(claim_check))
```

Consumer nhận URI, download từ Blob, process, optionally delete blob. Message nhỏ, không vượt limit.

Tại sao các đáp án còn lại sai:
- **Split thành 5 messages:** Phức tạp — cần track sequence, reassemble đúng thứ tự, handle partial failure (nếu 3/5 messages delivered). Không scalable, error-prone.
- **Base64 encode trên Premium:** Base64 encoding tăng size ~33%. 500 MB × 1.33 ≈ 665 MB — vẫn vượt 100 MB limit của Premium. Không giải quyết được.

---

## Câu 5

**You set the `correlation_id` property on every Service Bus message in your AI pipeline. What is the primary purpose of this property?**

- Tracking a request end-to-end across all pipeline stages, from the API through processing to result delivery ✅
- Enabling duplicate detection so Service Bus discards repeated submissions of the same request
- Routing messages to specific subscriptions based on filter rules

**Đáp án: End-to-end tracking qua toàn pipeline**

`correlation_id` là ID duy nhất follow request qua tất cả stages:

```
API enqueue → Service Bus → Worker process → Results store
      ↑              ↑            ↑               ↑
    correlation_id propagated everywhere
```

Troubleshoot failure: search logs, DLQ, result stores theo correlation_id → trace full lifecycle của request.

Tại sao các đáp án còn lại sai:
- **Duplicate detection:** Đó là `message_id`. Enable duplicate detection trên queue → Service Bus dùng `message_id` để discard duplicate submissions trong detection window.
- **Routing to subscriptions:** Đó là subscription **filter rules** (SQL filter, correlation filter). `correlation_id` không control routing trong Service Bus topology.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | 3 independent services cần react cùng message | Topic với 3 subscriptions |
| 2 | Không mất message khi worker crash | Peek-lock mode |
| 3 | Message fail 10 lần | Move to DLQ với MaxDeliveryCountExceeded |
| 4 | 500 MB file vượt 100 MB limit | Claim-check pattern (Blob URI) |
| 5 | correlation_id purpose | End-to-end tracking qua pipeline |

---

*Module hoàn thành.*

---

[← Bài 3](./backend-m1-bai3-process-messages.md) · [🏠 Mục lục](../README.md) · [Bài 1 →](./backend-m2-bai1-concepts.md)
