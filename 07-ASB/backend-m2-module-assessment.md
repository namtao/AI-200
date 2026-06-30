# Module Assessment — Develop Event-Driven AI Workflows with Azure Event Grid

> Khoá: AI-200 · Event Grid — Develop event-driven AI workflows with Azure Event Grid

---

## Câu 1

**You're building an AI content moderation application. You need to emit events when your inference service completes a classification. What type of Event Grid topic should you use to publish these events?**

- A custom topic ✅
- A system topic
- An Event Hubs topic

**Đáp án: Custom topic**

Custom topics là user-defined endpoints nơi **application code publishes events**. Inference completion là application-level event do code của bạn produce.

```bash
az eventgrid topic create \
    --name ai-pipeline-events \
    --resource-group ai-platform-rg \
    --location eastus \
    --input-schema cloudeventschemav1_0
```

Tại sao các đáp án còn lại sai:
- **System topic:** Event Grid tự tạo cho Azure service events (BlobCreated, KeyVaultKeyExpired...). **Only Azure service publishes** — application code không thể publish đến system topic. Dùng system topic để *react* to infrastructure events, không phải emit application events.
- **Event Hubs topic:** Không tồn tại trong Event Grid. Event Hubs là separate service cho high-throughput streaming (không phải event routing). Có thể là *destination* của Event Grid subscription, không phải topic type.

---

## Câu 2

**You're designing custom events for an AI pipeline and want to enable subscribers to filter events based on which pipeline stage produced them. Which CloudEvents attribute is best suited for path-based filtering with prefix and suffix matches?**

- The `subject` attribute ✅
- The `type` attribute
- The `source` attribute

**Đáp án: `subject`**

`subject` được thiết kế cho **hierarchical path-based filtering**. Set thành path phản ánh event context:

```
/pipelines/embeddings/batch-42
/pipelines/moderation/image-classifier
/models/classification/v3
```

Subscription filter:
```bash
--subject-begins-with /pipelines/embeddings    # Chỉ nhận events từ embeddings pipeline
--subject-ends-with /batch-42                  # Chỉ nhận events từ batch-42
```

Tại sao các đáp án còn lại sai:
- **`type`:** Categorize event category (`com.contoso.ai.InferenceCompleted`). Filter với `--included-event-types`. Không support prefix/suffix matching — chỉ exact match.
- **`source`:** Identify originating system (`/services/content-moderation`). Không có built-in prefix/suffix filter mechanism như subject. Subject là attribute được thiết kế cho fine-grained routing.

---

## Câu 3

**Your AI handler endpoint occasionally experiences cold-start latency that exceeds 30 seconds. You want to ensure events are still delivered successfully after the handler warms up. What should you do?**

- Rely on Event Grid's automatic retry mechanism with exponential backoff ✅
- Configure a dead-letter destination to capture timed-out events
- Set the event time-to-live to 30 seconds to match the timeout

**Đáp án: Rely on automatic retry with exponential backoff**

Cold-start timeout (30s exceeded) → Event Grid nhận no response → **retry automatically** với exponential backoff (10s → 30s → 1m → 5m...). Handler warm up trong khi Event Grid đang waiting → retry thành công.

Design best practice cho cold-start: return **202 Accepted** immediately và process async. Nhưng nếu function warm up trong retry window, automatic retry sẽ handle it.

Tại sao các đáp án còn lại sai:
- **Dead-letter destination:** Capture events **sau khi** all retries exhausted. Dead-lettering là last resort — không giúp deliver event trong khi handler warms up. Nếu handler warm up sau vài retries, automatic retry đã deliver thành công.
- **Set TTL to 30 seconds:** Ngược lại với mục tiêu! TTL 30 giây = event expire nhanh = Event Grid ngừng retry sau 30 giây. Cold start thường mất 10-20 giây → event đã expired trước khi handler ready.

---

## Câu 4

**You need to route AI moderation events to different handlers based on whether the content was flagged for review. The flagged status is stored in the `data.status` field of the event payload. Which filtering approach should you use?**

- Advanced filtering with the `StringIn` operator on `data.status` ✅
- Event type filtering with `--included-event-types`
- Subject filtering with `--subject-begins-with`

**Đáp án: Advanced filtering trên data.status**

Advanced filtering cho phép filter trên **values bên trong event body** (data fields):

```bash
az eventgrid event-subscription create \
    --name flagged-content-sub \
    --endpoint https://review-service.azurewebsites.net/api/events \
    --advanced-filter data.status StringIn flagged
```

`StringIn` operator match khi `data.status` là một trong các values trong list.

Tại sao các đáp án còn lại sai:
- **Event type filtering:** Filter trên `type` attribute (vd. `com.contoso.ai.ContentClassified`). Không có access đến data payload fields. Nếu tất cả events có cùng type, không thể distinguish flagged vs safe qua type filter.
- **Subject filtering:** Filter trên `subject` path — thường reflect pipeline/stage name, không phải processing outcome. Muốn filter theo outcome (flagged/safe), cần advanced filter trên data field.

---

## Câu 5

**You're deploying an AI inference service to Azure Functions and need to publish events to an Event Grid custom topic. What authentication approach should you use for production workloads?**

- Microsoft Entra ID with a managed identity assigned to the function app ✅
- Access key authentication using the `aeg-sas-key` header
- Store the access key in the function app's application settings

**Đáp án: Managed identity với Microsoft Entra ID**

```python
from azure.identity import DefaultAzureCredential
from azure.eventgrid import EventGridPublisherClient

credential = DefaultAzureCredential()    # Uses managed identity on Azure
client = EventGridPublisherClient(endpoint, credential)
```

Assign **Event Grid Data Sender** role cho function app's managed identity trên custom topic.

Lợi ích: không lưu secrets, không cần rotate keys, tích hợp với Conditional Access, audit trails.

Tại sao các đáp án còn lại sai:
- **Access key với `aeg-sas-key`:** Phù hợp cho development/testing và CI/CD pipelines. Không recommended cho production vì shared secret phải rotate thủ công, không có audit per-identity, nếu leaked = full access.
- **Store key in application settings:** Tốt hơn hardcode nhưng vẫn là shared secret. Nếu application settings bị compromised, key bị lộ. Managed identity loại bỏ hoàn toàn secret management.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Topic type cho application events | Custom topic |
| 2 | Path-based filtering theo pipeline stage | `subject` attribute |
| 3 | Handle cold-start timeout > 30s | Automatic retry với exponential backoff |
| 4 | Filter theo data.status field | Advanced filter `StringIn` |
| 5 | Auth cho production Azure Functions | Managed identity với Entra ID |

---

*Module hoàn thành.*

---

[← Bài 4](./backend-m2-bai4-publish-events.md) · [🏠 Mục lục](../README.md) · [Bài 1 →](./backend-m3-bai1-hosting-scaling.md)
