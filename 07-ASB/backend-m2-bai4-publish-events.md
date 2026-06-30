# Bài 4 — Publish Custom Events from AI Applications

> Khoá: AI-200 · Event Grid — Develop event-driven AI workflows with Azure Event Grid

---

## Tạo Custom Topic

```bash
# Tạo topic với CloudEvents schema
az eventgrid topic create \
    --name ai-pipeline-events \
    --resource-group ai-platform-rg \
    --location eastus \
    --input-schema cloudeventschemav1_0

# Get endpoint và key
az eventgrid topic show --name ai-pipeline-events -g ai-platform-rg --query "endpoint" -o tsv
az eventgrid topic key list --name ai-pipeline-events -g ai-platform-rg --query "key1" -o tsv
```

**Auth methods:**
- Access key (`aeg-sas-key` header) — development
- SAS token
- **Microsoft Entra ID** — **recommended production** (managed identity + Event Grid Data Sender role)

---

## Publish với Python SDK

### Single event

```python
from azure.core.credentials import AzureKeyCredential
from azure.core.messaging import CloudEvent
from azure.eventgrid import EventGridPublisherClient
import os

endpoint = os.environ["EVENTGRID_TOPIC_ENDPOINT"]
credential = AzureKeyCredential(os.environ["EVENTGRID_TOPIC_KEY"])
client = EventGridPublisherClient(endpoint, credential)

event = CloudEvent(
    type="com.contoso.ai.InferenceCompleted",
    source="/services/content-moderation",
    data={
        "requestId": "req-78901",
        "modelName": "content-classifier-v3",
        "processingDurationMs": 1250,
        "resultLocation": "https://results.blob.core.windows.net/output/req-78901.json",
        "status": "completed"
    },
    subject="/pipelines/moderation/image-classifier"
)

client.send(event)
```

### Production — Managed Identity

```python
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()    # Managed Identity trên Azure, CLI local
client = EventGridPublisherClient(endpoint, credential)
```

### Batch publish

```python
events = [
    CloudEvent(
        type="com.contoso.ai.StageCompleted",
        source="/services/embeddings",
        data={"pipelineRunId": "run-42", "stage": "embeddings", "status": "completed"},
        subject="/pipelines/rag/run-42"
    ),
    CloudEvent(
        type="com.contoso.ai.StageCompleted",
        source="/services/indexing",
        data={"pipelineRunId": "run-42", "stage": "indexing", "status": "completed"},
        subject="/pipelines/rag/run-42"
    )
]

client.send(events)    # Single HTTP request
```

---

## Publish với REST API

```bash
curl -X POST \
    -H "Content-Type: application/cloudevents+json; charset=utf-8" \
    -H "aeg-sas-key: $EVENTGRID_TOPIC_KEY" \
    -d '{
        "specversion": "1.0",
        "type": "com.contoso.ai.ModelRetrained",
        "source": "/services/training",
        "id": "evt-20250915-160000-001",
        "time": "2025-09-15T16:00:00Z",
        "subject": "/models/sentiment-v2",
        "datacontenttype": "application/json",
        "data": {
            "modelName": "sentiment-v2",
            "modelVersion": "2.1.0",
            "accuracy": 0.94,
            "trainingDurationMinutes": 45
        }
    }' \
    "$EVENTGRID_TOPIC_ENDPOINT"
```

REST returns **200 OK** when event accepted. Content-Type: `application/cloudevents+json`.

---

## Event Design Principles

```json
// Tốt: reference results, include correlation ID
{
    "data": {
        "requestId": "req-78901",
        "modelVersion": "3.2.1",
        "processingDurationMs": 1250,
        "resultLocation": "https://results.blob.core.windows.net/output/req-78901.json",
        "status": "completed",
        "summary": {"classification": "safe", "confidence": 0.97}
    }
}
// Không nên: embed full inference results
```

- `requestId` — correlation qua distributed services
- `resultLocation` — reference không embed full results
- `processingDurationMs`, `modelVersion` — operational metadata cho monitoring
- `summary` — fast routing decisions (flag for review?) không cần download full output

---

## Publish Points

Publish tại **natural checkpoint boundaries**:
- Inference complete
- Pipeline stage transition
- Anomaly detected
- Model validation complete

KHÔNG publish internal state changes mà no external subscriber cần biết.

---

## Bản chất bài này là gì?

**Một câu:** Publish custom events = tạo custom topic với CloudEvents schema, dùng `EventGridPublisherClient` + `DefaultAzureCredential` (managed identity production), thiết kế event payload nhỏ với reference thay vì embed full results.

### So sánh với AWS EventBridge / Kafka Producer / REST Webhook

| | Azure Event Grid publish | AWS EventBridge PutEvents | Kafka Producer | REST Webhook (push) |
|---|---|---|---|---|
| Auth (production) | Managed identity + Data Sender role | IAM role | SASL/mTLS | API key / OAuth |
| Auth (dev) | Access key header | AWS SDK credentials | Không | API key |
| SDK class | `EventGridPublisherClient` | `boto3 events` client | `KafkaProducer` | `requests.post()` |
| Batch | `client.send(list)` | Up to 10 entries / call | Producer batching | Không native |
| Max payload | 1 MB | 256 KB | 1 MB (default) | Tuỳ server |
| Pricing | Per event | Per event | Per throughput | Không |
| Schema validation | CloudEvents / EG Schema | EventBridge schema registry | Schema Registry | Không |

**Event design principle:** Payload nhỏ = fast routing + ít bandwidth. Include `resultLocation` (Blob URI) thay vì embed full inference output. Subscriber download khi cần — không phải mọi subscriber đều cần full output.

**Exam trap:** Managed identity cần **Event Grid Data Sender** role (không phải Contributor, không phải Owner). Đây là role tối thiểu để publish events — least privilege principle quan trọng trong exam.

---

## Checklist ghi nhớ cho AI-200

- [ ] Custom topic: `--input-schema cloudeventschemav1_0`
- [ ] Python SDK: `EventGridPublisherClient` + `CloudEvent`
- [ ] Auth: access key (dev) · **`DefaultAzureCredential` + managed identity** (production)
- [ ] Managed identity cần **Event Grid Data Sender** role trên topic
- [ ] REST API: `Content-Type: application/cloudevents+json` · `aeg-sas-key` header
- [ ] REST returns **200 OK** khi event accepted
- [ ] Batch: `client.send(list)` = single HTTP request
- [ ] Event design: reference results (blob URI), không embed · include `requestId` cho correlation
- [ ] Publish tại natural checkpoints, không phải internal state changes

---

*Module Assessment tiếp theo*

---

[← Bài 3](./backend-m2-bai3-delivery-retry.md) · [🏠 Mục lục](../README.md) · [Assessment →](./backend-m2-module-assessment.md)
