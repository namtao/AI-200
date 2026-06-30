# Bài 3 — Create Triggers and Bindings for AI Integration Patterns

> Khoá: AI-200 · Azure Functions — Build serverless AI backends with Azure Functions

---

## Triggers vs Bindings

- **Trigger:** Xác định cách function được invoke. Mỗi function có **đúng một** trigger. Trigger cũng là input binding (cung cấp event data).
- **Input bindings:** Data flow vào function
- **Output bindings:** Data flow ra khỏi function (optional, 0 đến nhiều)

Bindings xử lý authentication và serialization — code chỉ focus vào business logic. Services không có binding support → tạo SDK client trực tiếp.

---

## HTTP Trigger — Inference Endpoint

```python
import azure.functions as func
import json

app = func.FunctionApp()

@app.route(route="classify", methods=["POST"], auth_level=func.AuthLevel.FUNCTION)
def classify_document(req: func.HttpRequest) -> func.HttpResponse:
    payload = req.get_json()
    document_text = payload.get("text")

    classification = perform_classification(document_text)

    return func.HttpResponse(
        json.dumps({"category": classification}),
        status_code=200,
        mimetype="application/json"
    )
```

**Authorization levels:**
- `anonymous` — no key (health check, behind API gateway)
- **`function`** — function-specific key hoặc host key (**recommended production**)
- `admin` — master key only (administrative ops)

**HTTP timeout = 230 giây.** Tác vụ lâu hơn → return 202 + enqueue cho async processing.

---

## Service Bus Queue Trigger — Async Batch Processing

```python
@app.service_bus_queue_trigger(arg_name="msg", queue_name="document-jobs", connection="ServiceBusConnection")
def process_document(msg: func.ServiceBusMessage) -> None:
    job = json.loads(msg.get_body().decode("utf-8"))
    document_url = job["document_url"]
    job_id = job["job_id"]

    result = extract_and_classify(document_url)
    save_result(job_id, result)
```

**host.json config cho Service Bus:**

```json
{
    "version": "2.0",
    "extensions": {
        "serviceBus": {
            "maxConcurrentCalls": 1,        // 1 message per instance = full resources per message
            "maxAutoLockRenewalDuration": "00:05:00"  // Đủ lâu cho longest processing
        }
    }
}
```

**Dead-letter:** Khi message vượt `maxDeliveryCount` (default 10) trên queue resource → broker tự động move sang dead-letter subqueue với reason và description.

---

## Output Bindings

### Blob Storage output

```python
@app.service_bus_queue_trigger(arg_name="msg", queue_name="document-jobs", connection="ServiceBusConnection")
@app.blob_output(arg_name="output_blob", path="results/{rand-guid}.json", connection="AzureWebJobsStorage")
def process_and_store(msg: func.ServiceBusMessage, output_blob: func.Out[str]) -> None:
    job = json.loads(msg.get_body().decode("utf-8"))
    result = extract_and_classify(job["document_url"])

    output_blob.set(json.dumps({
        "job_id": job["job_id"],
        "classification": result["category"],
        "confidence": result["score"]
    }))
```

### Cosmos DB output

```python
@app.service_bus_queue_trigger(arg_name="msg", queue_name="classification-results", connection="ServiceBusConnection")
@app.cosmos_db_output(
    arg_name="output_doc",
    database_name="ai-results",
    container_name="classifications",
    connection="CosmosDBConnection"
)
def store_classification(msg: func.ServiceBusMessage, output_doc: func.Out[func.Document]) -> None:
    result = json.loads(msg.get_body().decode("utf-8"))

    output_doc.set(func.Document.from_dict({
        "id": result["job_id"],
        "category": result["category"],
        "confidence": result["score"]
    }))
```

---

## SDK Client cho Services Không Có Binding

Azure AI Document Intelligence, Azure OpenAI, Azure AI Search không có bindings → tạo SDK client trực tiếp.

**Khởi tạo ở module level** — persist across invocations, tránh overhead per-request:

```python
from azure.identity import DefaultAzureCredential
from azure.ai.documentintelligence import DocumentIntelligenceClient
import os

credential = DefaultAzureCredential()

doc_intelligence_client = DocumentIntelligenceClient(
    endpoint=os.environ["DOCUMENT_INTELLIGENCE_ENDPOINT"],
    credential=credential
)

@app.service_bus_queue_trigger(arg_name="msg", queue_name="document-jobs", connection="ServiceBusConnection")
def analyze_document(msg: func.ServiceBusMessage) -> None:
    job = json.loads(msg.get_body().decode("utf-8"))
    poller = doc_intelligence_client.begin_analyze_document(...)
    result = poller.result()
```

`DefaultAzureCredential`: local → Azure CLI credentials, production → managed identity. Same code, no changes.

---

## Bản chất bài này là gì?

**Một câu:** Triggers kích hoạt function (đúng 1 trigger), bindings xử lý I/O tự động (0-nhiều) — Azure AI services không có bindings nên khởi tạo SDK client ở module level để tránh per-request overhead.

### So sánh với AWS Lambda Event Sources / Google Cloud Triggers / Dapr

| | Azure Functions Bindings | AWS Lambda Event Sources | Google Cloud Triggers | Dapr Bindings |
|---|---|---|---|---|
| Trigger types | HTTP, Service Bus, Timer, Blob, EG, CosmosDB... | SQS, SNS, API GW, DynamoDB... | HTTP, Pub/Sub, Storage, Firestore... | HTTP, Kafka, RabbitMQ, Redis... |
| Output bindings | Blob, CosmosDB, Service Bus, EG, Table... | Không (explicit SDK calls) | Không | Output bindings |
| Auth abstraction | Có (managed identity auto) | IAM role (SDK handles) | Service account | Component config |
| SDK clients (AI services) | Module level (persist) | Global scope (persist) | Module level (persist) | External |
| Concurrency config | host.json serviceBus.maxConcurrentCalls | Event source batch size | Không | Component config |
| Dead-letter (trigger) | Queue maxDeliveryCount → auto DLQ | SQS redrive policy | Pub/Sub dead letter | Không |

**Tại sao module-level SDK client:** Azure Functions reuse instances across invocations — client khởi tạo một lần, kết nối được tái dụng. Nếu khởi tạo trong function handler → overhead connection setup mỗi invocation, latency tăng đáng kể với AI service calls.

**Exam trap:** `maxConcurrentCalls: 1` trong host.json serviceBus section = 1 message per instance. Đây là config cho AI workloads cần full resources. Đừng nhầm với `batchSize` (dành cho Storage Queue, không phải Service Bus).

---

## Checklist ghi nhớ cho AI-200

- [ ] Mỗi function có **đúng một** trigger · Input/output bindings optional
- [ ] HTTP trigger: `@app.route()` · Auth levels: anonymous, **function**, admin
- [ ] Service Bus trigger: `@app.service_bus_queue_trigger()`
- [ ] `maxConcurrentCalls: 1` = one message per instance = full resources per AI task
- [ ] `maxAutoLockRenewalDuration` phải đủ lâu cho longest processing
- [ ] Dead-letter triggered by `maxDeliveryCount` on queue resource (default 10)
- [ ] Blob output: `@app.blob_output()` · Cosmos DB output: `@app.cosmos_db_output()`
- [ ] SDK clients (AI services, OpenAI): khởi tạo ở **module level** (persist across invocations)
- [ ] `DefaultAzureCredential`: CLI local, managed identity production

---

[← Bài 2](./backend-m3-bai2-local-dev.md) · [🏠 Mục lục](../README.md) · [Bài 4 →](./backend-m3-bai4-secrets-config.md)
