# Bài 2 — Work with Event Schemas and Properties

> Khoá: AI-200 · Event Grid — Develop event-driven AI workflows with Azure Event Grid

---

## CloudEvents vs Event Grid Schema

| | CloudEvents v1.0 | Event Grid Schema |
|---|---|---|
| Standard | CNCF open standard | Azure proprietary |
| Interoperability | Any CloudEvents-compatible system | Azure only |
| **Recommendation** | **New implementations** | Backward compat |

**Chuyển đổi:** Event Grid Schema → CloudEvents OK. CloudEvents → Event Grid Schema **không support** (CloudEvents có extension attributes không map được).

---

## CloudEvents Structure

```json
{
    "specversion": "1.0",
    "type": "com.contoso.ai.InferenceCompleted",
    "source": "/services/content-moderation",
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "time": "2025-09-15T14:30:00Z",
    "subject": "/pipelines/moderation/batch-42",
    "datacontenttype": "application/json",
    "data": {
        "modelName": "content-classifier-v3",
        "requestId": "req-78901",
        "resultLocation": "https://storage.blob.core.windows.net/results/batch-42.json",
        "status": "success"
    }
}
```

### Required attributes

| Attribute | Mục đích |
|---|---|
| `specversion` | CloudEvents version — luôn `"1.0"` |
| `type` | Categorize event → drives event type filtering. **Reverse-DNS naming** |
| `source` | Originating system/component |
| `id` | Unique identifier → deduplicate repeated deliveries |

### Optional attributes

| Attribute | Mục đích |
|---|---|
| `subject` | **Hierarchical path for filtering** — prefix/suffix filter |
| `time` | When event occurred (UTC) |
| `datacontenttype` | Payload format (`application/json`) |
| `data` | Event payload (operation-specific details) |

---

## Event Type Design

Naming convention: **reverse-DNS** để tránh collision.

```
com.contoso.ai.InferenceCompleted
com.contoso.ai.EmbeddingsRefreshed
com.contoso.ai.AnomalyDetected
com.contoso.ai.ModelRetrained
com.contoso.ai.ContentClassified
```

**Design principles:**
- Payload nhỏ — include reference (blob URI), không embed full results
- Reference results thay vì embed: `"resultLocation": "https://..."`
- Operational metadata: `processingDurationMs`, `modelVersion`

---

## Event Type Filtering

```bash
az eventgrid event-subscription create \
    --name inference-handler-sub \
    --source-resource-id /subscriptions/{sub-id}/.../topics/ai-events \
    --endpoint https://inference-handler.azurewebsites.net/api/events \
    --included-event-types com.contoso.ai.InferenceCompleted
```

Multiple types: space-separated trong `--included-event-types`.

---

## Subject Filtering (Prefix/Suffix)

`subject` = hierarchical path. Filter với `subjectBeginsWith` hoặc `subjectEndsWith`.

```bash
# Chỉ nhận events từ embeddings pipeline
az eventgrid event-subscription create \
    --name embeddings-sub \
    --endpoint https://embeddings-handler.azurewebsites.net/api/events \
    --subject-begins-with /pipelines/embeddings
```

---

## Advanced Filtering (Data Attributes)

Filter trên values bên trong event body. Tối đa **25 filter conditions** per subscription. Multiple conditions: AND between conditions, OR within each condition's values.

```bash
# Chỉ nhận events với data.status = "flagged"
az eventgrid event-subscription create \
    --name flagged-content-sub \
    --endpoint https://review-service.azurewebsites.net/api/events \
    --advanced-filter data.status StringIn flagged
```

Operators: `StringContains`, `StringIn`, `NumberGreaterThan`, `BoolEquals`, `IsNotNull`

**AI use cases:**
- Confidence-based: `data.confidence NumberGreaterThan 0.9`
- Model-specific: `data.modelName StringIn gpt-4o`
- Status-based: `data.status StringIn flagged failed`

---

## Tạo Custom Topic với CloudEvents

```bash
az eventgrid topic create \
    --name ai-events \
    --resource-group ai-platform-rg \
    --location eastus \
    --input-schema cloudeventschemav1_0
```

---

## Bản chất bài này là gì?

**Một câu:** CloudEvents là chuẩn mở CNCF cho event schema — dùng thay Event Grid Schema cho new implementations vì interoperability, với 4 required fields và subject cho path-based filtering.

### So sánh với Event Grid Schema / AsyncAPI / Custom JSON

| | CloudEvents v1.0 | Event Grid Schema | AsyncAPI | Custom JSON |
|---|---|---|---|---|
| Standard | CNCF open standard | Azure proprietary | API description standard | Không |
| Required fields | specversion, type, source, id | id, eventType, subject, data, eventTime | Varies | Tùy định nghĩa |
| Interoperability | Any CloudEvents system | Azure only | N/A | Không |
| Path filtering | `subject` (prefix/suffix) | `subject` | N/A | Không |
| Data filtering | Advanced filter trên `data.*` | Advanced filter | N/A | Không |
| Conversion | → EG Schema OK | → CloudEvents không support | N/A | N/A |
| SDK support | `CloudEvent` class | `EventGridEvent` class | N/A | Raw dict |

**Tại sao CloudEvents → EG Schema không support:** CloudEvents có extension attributes (custom fields) không map được sang Event Grid Schema format — thông tin bị mất.

**Exam trap:** `subject` dùng cho prefix/suffix path filtering (`--subject-begins-with`). `type` dùng cho exact match filtering (`--included-event-types`). Không thể dùng subject filter cho exact type match hoặc ngược lại.

---

## Checklist ghi nhớ cho AI-200

- [ ] CloudEvents = CNCF open standard, **recommended cho new implementations**
- [ ] CloudEvents → EG Schema **không support** · EG Schema → CloudEvents OK
- [ ] `specversion`: always `"1.0"` · `type`: reverse-DNS naming · `id`: for deduplication
- [ ] `subject` = hierarchical path → **prefix/suffix filtering**
- [ ] `--included-event-types` = event type filter
- [ ] `--subject-begins-with` / `--subject-ends-with` = subject filter
- [ ] `--advanced-filter data.field operator value` = data attribute filter
- [ ] Max **25** advanced filter conditions per subscription
- [ ] Keep payload small — reference results, don't embed

---

[← Bài 1](./backend-m2-bai1-concepts.md) · [🏠 Mục lục](../README.md) · [Bài 3 →](./backend-m2-bai3-delivery-retry.md)
