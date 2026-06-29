# Bài 5 — Configure Identity and Access for Functions

> Khoá: AI-200 · Azure Functions — Build serverless AI backends with Azure Functions

---

## Identity-based Connections

Thay connection strings bằng managed identity — loại bỏ secrets khỏi config hoàn toàn.

### AzureWebJobsStorage với identity

```
AzureWebJobsStorage__accountName = mystorageaccount
```

Runtime tự authenticate bằng managed identity. **Minimum role: `Storage Blob Data Owner`**.

Nếu dùng blob triggers: cũng cần `Storage Queue Data Contributor` và `Storage Account Contributor`.

### Service Bus với identity

```
ServiceBusConnection__fullyQualifiedNamespace = mynamespace.servicebus.windows.net
```

Roles:
- Trigger (receive): **`Azure Service Bus Data Receiver`**
- Output binding (send): **`Azure Service Bus Data Sender`**

```python
@app.service_bus_queue_trigger(
    arg_name="msg",
    queue_name="document-jobs",
    connection="ServiceBusConnection"    # Points đến env var prefix
)
def process_document(msg: func.ServiceBusMessage) -> None:
    ...
```

### Cosmos DB với identity

```
CosmosDBConnection__accountEndpoint = https://mycosmosaccount.documents.azure.com:443/
```

Role: **`Cosmos DB Built-in Data Contributor`**

---

## Suffix Pattern

| Service | Setting suffix | Example |
|---|---|---|
| Storage | `__accountName` | `AzureWebJobsStorage__accountName = myaccount` |
| Service Bus | `__fullyQualifiedNamespace` | `ServiceBusConnection__fullyQualifiedNamespace = ns.servicebus.windows.net` |
| Cosmos DB | `__accountEndpoint` | `CosmosDBConnection__accountEndpoint = https://account.documents.azure.com:443/` |

---

## DefaultAzureCredential cho SDK Clients

```python
import os
from azure.identity import DefaultAzureCredential
from azure.ai.documentintelligence import DocumentIntelligenceClient
from azure.search.documents import SearchClient

# Module-level — persist across invocations
credential = DefaultAzureCredential()

doc_client = DocumentIntelligenceClient(
    endpoint=os.environ["DOCUMENT_INTELLIGENCE_ENDPOINT"],
    credential=credential
)

search_client = SearchClient(
    endpoint=os.environ["SEARCH_ENDPOINT"],
    index_name=os.environ["SEARCH_INDEX"],
    credential=credential
)
```

`DefaultAzureCredential`: local → Azure CLI/VS Code credentials · production → managed identity.

**Required roles:**
- Azure AI Document Intelligence: `Cognitive Services User`
- Azure AI Search: `Search Index Data Reader` hoặc `Search Index Data Contributor`
- Azure OpenAI: `Cognitive Services OpenAI User`

---

## HTTP Access Keys

4 loại keys:

| Key type | Scope | Dùng khi |
|---|---|---|
| **Function key** | Single function | Protect individual AI endpoint |
| **Host key** | All functions in app | Admin tools, monitoring |
| **System keys** | Extension-specific | MCP extension (`mcp_extension`) |
| **Master key** | Admin + runtime APIs | Administrative ops only — never share |

```python
# Production inference endpoint
@app.route(route="classify", methods=["POST"], auth_level=func.AuthLevel.FUNCTION)
def classify_document(req: func.HttpRequest) -> func.HttpResponse:
    pass

# Health check — no key needed
@app.route(route="health", methods=["GET"], auth_level=func.AuthLevel.ANONYMOUS)
def health_check(req: func.HttpRequest) -> func.HttpResponse:
    return func.HttpResponse("OK", status_code=200)
```

Call: `x-functions-key` header hoặc `code` query parameter.

**Access keys = basic barrier, không replace identity verification.** Layer với API Management hoặc Easy Auth (Entra ID) cho production.

---

## Bản chất bài này là gì?

**Một câu:** Identity-based connections = managed identity thay connection strings qua suffix pattern (`__accountName`, `__fullyQualifiedNamespace`, `__accountEndpoint`) + đúng RBAC role cho từng service — loại bỏ hoàn toàn secrets khỏi config.

### So sánh với Connection Strings / AWS IAM / Service Principal

| | Azure Managed Identity | Connection Strings | Service Principal + Secret | AWS IAM Role |
|---|---|---|---|---|
| Secrets trong config | Không | Có | Có (client secret) | Không |
| Rotation | Tự động (platform managed) | Thủ công | Thủ công | Tự động |
| Audit trail | Per-identity | Không | Per-principal | Per-role |
| Local dev | Azure CLI / VS Code credentials | Same string | Env var | AWS credentials |
| Binding config pattern | `Connection__suffix = value` | Full connection string | Client ID + secret | N/A |
| Scope | Azure resources | Per service | Azure resources | AWS resources |

**RBAC roles quan trọng theo service:**

| Service | Trigger/Receive | Send/Write |
|---|---|---|
| Service Bus | Azure Service Bus Data Receiver | Azure Service Bus Data Sender |
| Storage | Storage Blob Data Owner (min) | Storage Blob Data Contributor |
| Cosmos DB | Cosmos DB Built-in Data Reader | Cosmos DB Built-in Data Contributor |
| Event Grid | — | Event Grid Data Sender |
| AI Document Intelligence | Cognitive Services User | — |
| Azure OpenAI | Cognitive Services OpenAI User | — |

**Exam trap:** Service Bus trigger cần **Data Receiver** (không phải Data Owner, không phải Contributor). Data Owner là overkill — least privilege principle. Key Vault Secrets User role là cho Key Vault, không apply sang Service Bus namespace — đây là distractor phổ biến trong assessment.

---

## Checklist ghi nhớ cho AI-200

- [ ] Identity-based: managed identity thay connection strings — no secrets in config
- [ ] `Storage Blob Data Owner` = minimum cho `AzureWebJobsStorage` identity
- [ ] Service Bus receive → `Azure Service Bus Data Receiver`
- [ ] Service Bus send → `Azure Service Bus Data Sender`
- [ ] Cosmos DB → `Cosmos DB Built-in Data Contributor`
- [ ] Suffix: `__accountName`, `__fullyQualifiedNamespace`, `__accountEndpoint`
- [ ] `DefaultAzureCredential`: CLI local, managed identity production — same code
- [ ] SDK clients khởi tạo ở **module level** — persist across invocations
- [ ] 4 key types: function, host, system, master
- [ ] `function` auth level = recommended cho production AI endpoints
- [ ] Access keys ≠ identity verification — layer với APIM/Easy Auth cho production

---

*Module Assessment tiếp theo*
