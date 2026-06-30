# Tổng Hợp — Azure Service Bus, Event Grid, Azure Functions

> Khoá: AI-200 · 07-ASB · 3 modules: M1 Service Bus · M2 Event Grid · M3 Azure Functions

---

## Bức Tranh 3 Modules

```
┌─────────────────────────────────────────────────────────────────────┐
│                      AI Application Flow                             │
│                                                                       │
│  User Request                                                         │
│      ↓                                                                │
│  [M3] Azure Functions (HTTP trigger)                                  │
│      → validate + enqueue → return 202                                │
│      ↓                                                                │
│  [M1] Service Bus Queue                                               │
│      → peek-lock delivery                                             │
│      ↓                                                                │
│  [M3] Azure Functions (Service Bus trigger)                           │
│      → run AI inference (Document Intelligence, OpenAI...)            │
│      → complete_message() on success                                  │
│      → publish result event                                           │
│      ↓                                                                │
│  [M2] Event Grid Custom Topic                                         │
│      → fan-out notifications                                          │
│      → notification handler / audit handler / dashboard handler       │
│                                                                       │
│  [M1] Service Bus Topic (nếu cần guaranteed delivery fan-out)        │
│      → topic + subscriptions cho reliable multi-consumer              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Module 1 — Azure Service Bus: Queue và Process AI Operations

### Core Concepts

**Queue:** Single-consumer. Mỗi message → đúng một consumer (competing consumers). Scale bằng cách thêm workers.

**Topic + Subscriptions:** Fan-out. Message copy đến TẤT CẢ subscriptions. Mỗi subscription = independent virtual queue. Add subscription mới không cần sửa producer.

**Receive modes:**
- **Peek-lock** (default): Lock message, consumer phải settle. Worker crash → lock expire → message available lại. At-least-once delivery.
- **Receive-and-delete**: Remove ngay khi deliver. Có thể mất message nếu crash.

**Settlement operations (Peek-lock):**

| Method | Effect |
|---|---|
| `complete_message()` | Remove permanently — success |
| `abandon_message()` | Release lock, requeue — transient error |
| `dead_letter_message()` | Move to DLQ — permanent error |
| `defer_message()` | Ẩn khỏi delivery, access bằng sequence number |

**DLQ:** Automatic subqueue. Max delivery count default = 10 → exceed = `MaxDeliveryCountExceeded`. Đọc bằng `sub_queue=ServiceBusSubQueue.DEAD_LETTER`.

**AutoLockRenewer:** Lock max 5 phút — long AI inference cần renew.

### Message Structure

- **Body**: JSON payload
- **Application properties**: Custom metadata (routing, tracking)
- **System properties**: `message_id`, `correlation_id`, `content_type`, `time_to_live`, `session_id`

**Limits:** Standard: 256 KB · Premium: 100 MB

**Claim-check pattern:** Large payload → upload Blob → send URI trong message → consumer download.

**message_id** = duplicate detection · **correlation_id** = end-to-end tracking

### Subscription Filters

- **SQL filter:** `priority = 'high'`, `model_name = 'gpt-4o'`
- **Correlation filter:** Exact-match, hiệu quả hơn SQL
- **Boolean filter:** TrueFilter / FalseFilter

---

## Module 2 — Azure Event Grid: Event-Driven AI Workflows

### Core Concepts

**Event Grid** = push-based event router. Per-event pricing. Sub-second latency. Không polling.

**5 components:** Events · Sources · Topics · Subscriptions · Handlers

**Topic types:**

| | System Topics | Custom Topics | Namespace Topics |
|---|---|---|---|
| Ai tạo | Event Grid tự tạo | User tạo | Part of namespace |
| Ai publish | Azure service | Your application | Your application |
| Dùng khi | Infrastructure events | App-level events | Push + pull, MQTT |

**Handlers:** Azure Functions · Event Hubs · Service Bus · Webhooks · Storage queues

### CloudEvents Schema

```json
{
    "specversion": "1.0",
    "type": "com.contoso.ai.InferenceCompleted",
    "source": "/services/content-moderation",
    "id": "a1b2c3d4-...",
    "subject": "/pipelines/moderation/batch-42",
    "data": { "resultLocation": "https://...", "status": "success" }
}
```

**Required:** `specversion`, `type`, `source`, `id`

**subject** = hierarchical path → prefix/suffix filtering (`--subject-begins-with`, `--subject-ends-with`)

**type** = reverse-DNS naming → exact match filtering (`--included-event-types`)

**Advanced filter:** Filter trên `data.*` fields. Max 25 conditions per subscription. Operators: `StringIn`, `StringContains`, `NumberGreaterThan`, `BoolEquals`.

**Schema conversion:** EG Schema → CloudEvents OK · CloudEvents → EG Schema không support.

### Delivery và Retry

**Success codes:** 200, 201, 202, 203, 204 · Timeout: 30 giây

**No-retry codes:** 400, 413, 403

**Exponential backoff:** 10s → 30s → 1m → 5m → 10m → 30m → 1h → 3h → 6h → 12h

**Retry config:**
- `--max-delivery-attempts` (1-30, default 30)
- `--event-ttl` (1-1440 min, default 1440)
- Event Grid stops khi đến limit nào trước

**Dead-letter:** Disabled by default. Cần Blob container. Reasons: `MaxDeliveryAttemptsExceeded` · `MaxRetryDurationExceeded`.

**At-least-once** → handlers phải idempotent, dùng `id` để deduplicate.

**Cold start:** Return 202 ngay, process async — tránh exceed 30s timeout.

### Publishing Events

```python
from azure.eventgrid import EventGridPublisherClient
from azure.core.messaging import CloudEvent
from azure.identity import DefaultAzureCredential

client = EventGridPublisherClient(endpoint, DefaultAzureCredential())
client.send(CloudEvent(type="com.contoso.ai.InferenceCompleted", source="/services/...", data={...}))
```

**Auth production:** Managed identity + **Event Grid Data Sender** role

---

## Module 3 — Azure Functions: Serverless AI Backends

### Hosting Plans

| Plan | Cold start | Scale | Dùng khi |
|---|---|---|---|
| **Flex Consumption** | Có (always-ready giảm) | Per-function, to 1000 instances | **Default mới — Linux only** |
| **Premium** | Không (pre-warmed) | Per-app | Continuous traffic, no cold start |
| Dedicated | N/A | Manual/auto | Underutilized App Service |
| Container Apps | N/A | Custom | GPU, custom OS |
| Consumption (Linux) | Có | Per-app | **Retire Sept 2028** |

**Always-ready instances (Flex):** Giữ N warm instances cho specific functions. Trả baseline + không trả cho zero-scale functions khác.

**Memory:** 512 MB · **2,048 MB (default, 1 vCPU)** · 4,096 MB (2 vCPU)

**HTTP timeout = 230 giây** (Azure Load Balancer, không override được). AI tasks > 230s → return 202 + enqueue.

### Local Development

**Tools:** Azure Functions Core Tools v4 + VS Code + Azurite

**Project structure:**
- `function_app.py` — entry point, tất cả functions + decorators
- `host.json` — global runtime config (timeout, logging, concurrency)
- `local.settings.json` — local secrets (**KHÔNG commit**)
- `.vscode/` — debug config (**nên commit**)

**Azurite:** `AzureWebJobsStorage = "UseDevelopmentStorage=true"` — bắt buộc cho non-HTTP triggers.

```bash
func start                                          # Start local
func azure functionapp fetch-app-settings <name>    # Download settings từ Azure
func settings encrypt                               # Encrypt secrets on dev machine
```

### Triggers và Bindings

**Trigger:** Đúng 1 per function. Xác định cách invoke.

**Bindings:** 0-nhiều input/output. Xử lý auth + serialization tự động.

**HTTP trigger auth levels:** `anonymous` · **`function`** (recommended) · `admin`

**Service Bus trigger host.json:**
```json
{
    "extensions": {
        "serviceBus": {
            "maxConcurrentCalls": 1,
            "maxAutoLockRenewalDuration": "00:05:00"
        }
    }
}
```

**SDK clients (AI services không có bindings):** Khởi tạo ở **module level** — persist across invocations.

### Secrets và Configuration

**3 tầng:**
1. App settings → `os.environ["KEY"]` — encrypted at rest
2. Key Vault references → `@Microsoft.KeyVault(VaultName=...;SecretName=...)` — managed identity + **Key Vault Secrets User** role
3. App Configuration → centralized non-secret config, labels cho multi-env

**Versionless URI** → auto-rotate 24h. **Versioned URI** → pinned, cần update khi rotate.

### Identity và Access

**Identity-based connections (suffix pattern):**

| Service | Suffix | Example |
|---|---|---|
| Storage | `__accountName` | `AzureWebJobsStorage__accountName = myaccount` |
| Service Bus | `__fullyQualifiedNamespace` | `ServiceBusConnection__fullyQualifiedNamespace = ns.servicebus.windows.net` |
| Cosmos DB | `__accountEndpoint` | `CosmosDBConnection__accountEndpoint = https://account.documents.azure.com:443/` |

**RBAC roles:**

| Service | Role cần thiết |
|---|---|
| Service Bus trigger (receive) | **Azure Service Bus Data Receiver** |
| Service Bus output (send) | **Azure Service Bus Data Sender** |
| Storage (AzureWebJobsStorage) | **Storage Blob Data Owner** |
| Cosmos DB | **Cosmos DB Built-in Data Contributor** |
| Event Grid publish | **Event Grid Data Sender** |
| Key Vault secrets | **Key Vault Secrets User** |
| Azure OpenAI | **Cognitive Services OpenAI User** |
| AI Document Intelligence | **Cognitive Services User** |

**HTTP keys:** function · host · system · master. `function` level = recommended cho production endpoints.

---

## CLI / Code Quick Reference

### Service Bus

```bash
# Tạo namespace
az servicebus namespace create --name myns --resource-group myrg --sku Standard

# Tạo queue
az servicebus queue create --name inference-requests --namespace-name myns -g myrg

# Tạo topic + subscription
az servicebus topic create --name inference-results --namespace-name myns -g myrg
az servicebus topic subscription create --name notifications --topic-name inference-results --namespace-name myns -g myrg

# Add SQL filter rule
az servicebus topic subscription rule create --name high-priority \
    --subscription-name high-priority-sub --topic-name inference-results \
    --namespace-name myns -g myrg --filter-sql-expression "priority = 'high'"
```

```python
# Send
with ServiceBusClient(namespace, DefaultAzureCredential()) as client:
    with client.get_queue_sender("inference-requests") as sender:
        sender.send_messages(ServiceBusMessage(json.dumps(payload), message_id=str(uuid4())))

# Receive (peek-lock)
with client.get_queue_receiver("inference-requests", receive_mode=PEEK_LOCK) as receiver:
    for msg in receiver:
        try:
            process(msg); receiver.complete_message(msg)
        except PermanentError:
            receiver.dead_letter_message(msg, reason="...", error_description="...")
        except TransientError:
            receiver.abandon_message(msg)

# Read DLQ
with client.get_queue_receiver("inference-requests", sub_queue=ServiceBusSubQueue.DEAD_LETTER) as dlq:
    for msg in dlq:
        print(msg.dead_letter_reason, msg.dead_letter_error_description)
        dlq.complete_message(msg)
```

### Event Grid

```bash
# Tạo custom topic
az eventgrid topic create --name ai-pipeline-events -g myrg --location eastus \
    --input-schema cloudeventschemav1_0

# Tạo subscription với filters
az eventgrid event-subscription create \
    --name inference-sub \
    --source-resource-id /subscriptions/.../topics/ai-pipeline-events \
    --endpoint https://handler.azurewebsites.net/api/events \
    --included-event-types com.contoso.ai.InferenceCompleted \
    --subject-begins-with /pipelines/moderation \
    --advanced-filter data.status StringIn flagged \
    --max-delivery-attempts 5 \
    --event-ttl 60 \
    --deadletter-endpoint /subscriptions/.../storageAccounts/myaccount/blobServices/default/containers/dlq

# Assign Event Grid Data Sender role
az role assignment create \
    --assignee <managed-identity-object-id> \
    --role "EventGrid Data Sender" \
    --scope /subscriptions/.../topics/ai-pipeline-events
```

```python
# Publish event
from azure.eventgrid import EventGridPublisherClient
from azure.core.messaging import CloudEvent

client = EventGridPublisherClient(endpoint, DefaultAzureCredential())
client.send(CloudEvent(
    type="com.contoso.ai.InferenceCompleted",
    source="/services/moderation",
    subject="/pipelines/moderation/batch-42",
    data={"requestId": "req-001", "resultLocation": "https://...", "status": "completed"}
))
```

### Azure Functions

```bash
# Init project
func init my-ai-backend --python

# New function
func new --name classify --template "HTTP trigger" --authlevel function
func new --name process-doc --template "Service Bus Queue Trigger"

# Run local
func start

# Deploy
func azure functionapp publish <app-name>

# Fetch settings
func azure functionapp fetch-app-settings <app-name>
```

```python
import azure.functions as func
import json, os
from azure.identity import DefaultAzureCredential
from azure.ai.documentintelligence import DocumentIntelligenceClient

app = func.FunctionApp()

# Module-level SDK client
credential = DefaultAzureCredential()
doc_client = DocumentIntelligenceClient(os.environ["DOC_INTELLIGENCE_ENDPOINT"], credential)

# HTTP trigger (accept + enqueue pattern)
@app.route(route="process-document", methods=["POST"], auth_level=func.AuthLevel.FUNCTION)
@app.service_bus_queue_output(arg_name="queue_msg", queue_name="document-jobs", connection="ServiceBusConnection")
def accept_document(req: func.HttpRequest, queue_msg: func.Out[str]) -> func.HttpResponse:
    job_id = str(uuid.uuid4())
    queue_msg.set(json.dumps({"job_id": job_id, "document_url": req.get_json().get("document_url")}))
    return func.HttpResponse(json.dumps({"job_id": job_id}), status_code=202, mimetype="application/json")

# Service Bus trigger
@app.service_bus_queue_trigger(arg_name="msg", queue_name="document-jobs", connection="ServiceBusConnection")
def process_document(msg: func.ServiceBusMessage) -> None:
    job = json.loads(msg.get_body().decode("utf-8"))
    result = doc_client.begin_analyze_document(...).result()
    save_result(job["job_id"], result)
```

```json
// host.json — Service Bus AI config
{
    "version": "2.0",
    "extensions": {
        "serviceBus": {
            "maxConcurrentCalls": 1,
            "maxAutoLockRenewalDuration": "00:10:00"
        }
    }
}

// local.settings.json (KHÔNG commit)
{
    "IsEncrypted": false,
    "Values": {
        "AzureWebJobsStorage": "UseDevelopmentStorage=true",
        "FUNCTIONS_WORKER_RUNTIME": "python",
        "ServiceBusConnection__fullyQualifiedNamespace": "myns.servicebus.windows.net"
    }
}
```

---

## So Sánh Service Bus vs Event Grid vs Event Hubs vs Redis Streams

| | Service Bus | Event Grid | Event Hubs | Redis Streams |
|---|---|---|---|---|
| **Mục đích** | Message broker, guaranteed delivery | Event router, lightweight notifications | High-throughput streaming | In-memory streaming |
| **Model** | Queue / Topic+Subscriptions | Push subscribe (HTTP) | Partitioned log | Consumer groups |
| **Ordering** | Sessions = FIFO per group | Không đảm bảo | Per-partition ordering | Per-stream ordering |
| **Retention** | Configurable (days) | 24h retry window | 1-90 ngày | Memory limit |
| **Max message size** | 256 KB (Standard) / 100 MB (Premium) | 1 MB | 1 MB | Không giới hạn (practical) |
| **Delivery guarantee** | At-least-once (peek-lock) | At-least-once | At-least-once | At-least-once (ACK) |
| **Dead-letter** | Automatic subqueue | Blob container (opt-in) | Không native | Không native |
| **Filter** | SQL filter, Correlation filter | Event type, subject, advanced (data) | Không | Không |
| **Fan-out** | Topic + subscriptions | Event subscriptions | Consumer groups | Consumer groups |
| **Pricing** | Per operation | Per event | Per throughput unit | Per compute/memory |
| **Latency** | Low (ms) | Sub-second | Very low (ms) | Sub-ms |
| **Managed** | Fully (Azure) | Fully (Azure) | Fully (Azure) | Self-hosted / Redis Cloud |
| **AI use case** | Inference job queue, reliable async processing | Pipeline stage events, infrastructure reactions | High-volume telemetry, streaming ingestion | Real-time session state, rate limiting |

**Khi nào dùng gì:**
- **Service Bus Queue:** AI inference jobs cần at-least-once, retry, DLQ, ordering (sessions)
- **Service Bus Topic:** Same event → multiple independent consumers (notification + audit + metrics)
- **Event Grid:** "Something happened" lightweight notifications, pipeline stage coordination, infrastructure events (BlobCreated)
- **Event Hubs:** High-throughput telemetry ingestion, streaming ML pipeline input
- **Redis Streams:** Real-time session context, sub-ms latency requirement, ephemeral data

---

## Exam Traps (≥10)

### Trap 1: Queue vs Topic fan-out
Queue competing consumers = chỉ MỘT worker nhận mỗi message. Topic subscriptions = TẤT CẢ subscriptions nhận copy. Nếu 3 services cần cùng message → Topic, không phải 3 competing consumers.

### Trap 2: message_id vs correlation_id
`message_id` = duplicate detection (Service Bus dùng để discard duplicates trong detection window). `correlation_id` = end-to-end tracking qua pipeline stages. Không thể hoán đổi — sai sẽ mất duplicate detection hoặc không trace được.

### Trap 3: Lock max duration
Lock max = **5 phút** (default setting thấp hơn). AI inference > 5 phút mà không dùng `AutoLockRenewer` → Service Bus làm available lại → duplicate processing. Không phải "message bị xóa" hay "message vào DLQ".

### Trap 4: Event Grid dead-letter disabled by default
Service Bus DLQ tự động tồn tại (luôn có). Event Grid dead-letter **disabled by default**, phải explicitly configure `--deadletter-endpoint` trỏ Blob container. Không config = events mất vĩnh viễn sau retry exhaustion.

### Trap 5: System topic vs Custom topic
Application code KHÔNG THỂ publish đến system topics (chỉ Azure services publish). Để emit custom application events → tạo Custom Topic. System topics dùng để react to Azure infrastructure events.

### Trap 6: CloudEvents → Event Grid Schema conversion
CloudEvents → EG Schema **không support** (extension attributes không map được). EG Schema → CloudEvents OK. Luôn dùng CloudEvents cho new implementations.

### Trap 7: HTTP timeout 230 giây
Azure Load Balancer timeout = 230 giây, **không thể override** bằng `functionTimeout` trong host.json. AI inference > 230s PHẢI dùng async pattern: return 202, enqueue job, separate worker processes.

### Trap 8: Consumption (Linux) retire
Consumption plan Linux sẽ **retire September 2028**. Câu hỏi về "new AI function app" → Flex Consumption, không phải Consumption. Premium khi cần eliminate cold start hoàn toàn.

### Trap 9: maxConcurrentCalls vs batchSize
`maxConcurrentCalls` = Service Bus trigger concurrency config (trong host.json `serviceBus` section). `batchSize` = Storage Queue trigger. Nhầm hai cái này = không apply config đúng service.

### Trap 10: Service Bus trigger role
Identity-based Service Bus trigger cần **Azure Service Bus Data Receiver** (receive only). Data Owner là quá broad. Key Vault Secrets User không apply cho Service Bus namespace — distractor phổ biến.

### Trap 11: Versionless URI rotation timing
Versionless Key Vault URI auto-rotate trong **24 giờ** (không phải ngay). Nếu cần immediate rotation: config change → trigger immediate refetch. Versioned URI không auto-rotate, cần update app setting → effective redeploy.

### Trap 12: local.settings.json vs .vscode/
`local.settings.json` = KHÔNG commit (chứa connection strings, secrets). `.vscode/` = nên commit (team chia sẻ debug config). Non-HTTP triggers không fire nếu `AzureWebJobsStorage` không được configure với Azurite.

### Trap 13: Event Grid retry — cold start
Cold start > 30s → Event Grid nhận timeout → **auto retry với exponential backoff** (đây là correct behavior). Dead-letter không giúp deliver event trong khi handler warms up. TTL 30s = ngược lại — event expire trước khi handler ready.

### Trap 14: Advanced filter vs subject filter vs event type filter
- `data.status = "flagged"` → advanced filter (`--advanced-filter data.status StringIn flagged`)
- Filter theo pipeline stage path → subject filter (`--subject-begins-with /pipelines/embeddings`)
- Filter theo event category → event type filter (`--included-event-types com.contoso.ai.InferenceCompleted`)
- Event type filter KHÔNG có access đến data payload fields.

---

## Assessment Summaries

### M1 Assessment — Service Bus (5/5)

| Câu | Scenario | Đáp án |
|---|---|---|
| 1 | 3 services cần react cùng message | **Topic với 3 subscriptions** (không phải 3 competing consumers) |
| 2 | Không mất message khi worker crash | **Peek-lock** (receive-and-delete = mất message) |
| 3 | Message fail 10 lần | **Move to DLQ với MaxDeliveryCountExceeded** (không xóa, không retry mãi) |
| 4 | 500 MB file vượt 100 MB Premium limit | **Claim-check pattern** (Base64 tăng size thêm 33% = vẫn vượt limit) |
| 5 | correlation_id purpose | **End-to-end tracking** (không phải duplicate detection — đó là message_id) |

### M2 Assessment — Event Grid (5/5)

| Câu | Scenario | Đáp án |
|---|---|---|
| 1 | Application emit inference completion events | **Custom topic** (system topic = Azure service publishes, không phải app code) |
| 2 | Filter theo pipeline stage với prefix/suffix | **`subject` attribute** (type = exact match, source = không có prefix/suffix filter) |
| 3 | Cold-start > 30s, đảm bảo delivery | **Automatic retry với exponential backoff** (dead-letter là sau retry exhaustion, TTL 30s = ngược lại) |
| 4 | Route theo data.status = "flagged" | **Advanced filter `StringIn` trên `data.status`** (event type filter không access data fields) |
| 5 | Auth production publish từ Functions | **Managed identity + Event Grid Data Sender role** (access key = dev only) |

### M3 Assessment — Azure Functions (5/5)

| Câu | Scenario | Đáp án |
|---|---|---|
| 1 | Reduce cold start, keep costs low | **Flex Consumption + always-ready instances** (Premium = higher baseline, Consumption = retire 2028) |
| 2 | Service Bus trigger không fire locally | **AzureWebJobsStorage không configure** (không cần deploy lên Azure để test trigger) |
| 3 | One message per instance, full resources | **`maxConcurrentCalls: 1` trong host.json** (batchSize = Storage Queue, maxDequeueCount = delivery attempts) |
| 4 | Key Vault, rotate without redeploy | **Versionless secret URI** (versioned = phải update setting khi rotate = effective redeploy) |
| 5 | Identity-based Service Bus trigger | **Azure Service Bus Data Receiver** (Data Owner = overkill, Key Vault Secrets User = wrong service) |

---

*Tổng hợp hoàn chỉnh 07-ASB — 3 modules · 12 bài học · 15 files.*

---

[← Assessment](./backend-m3-module-assessment.md) · [🏠 Mục lục](../README.md) · [Mục lục →](../README.md)
