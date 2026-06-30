# Bài 4 — Use the Change Feed to Trigger Embedding Refresh

> Khoá: AI-200 · Cosmos DB — Implement vector search on Azure Cosmos DB for NoSQL

---

## Vấn đề: Stale Embeddings

Khi document content thay đổi nhưng embedding không được regenerate → embedding không còn đại diện đúng cho content → search trả về kết quả không chính xác.

**Change feed** giải quyết: tự động detect document change và trigger embedding regeneration — không cần polling.

---

## Change Feed là gì?

Change feed là **persistent, ordered log** của tất cả insert và update trong container.

**Đặc điểm:**
- **Enabled mặc định** — tất cả container, không cần cấu hình thêm
- **Persistent** — changes được retain, có thể process kể cả khi app offline
- **Ordered delivery** — changes trong cùng partition key theo thứ tự modification
- **Exactly-once opportunity** — mỗi change xuất hiện đúng một lần (consumer tự handle checkpointing)

> **Lưu ý:** Change feed **không capture deletes** — chỉ insert và update. Nếu cần track delete, dùng soft-delete pattern (set `isDeleted: true` thay vì xoá thực sự).

---

## Push Model vs Pull Model

### Push Model (recommended cho production)

Platform tự deliver changes đến code của bạn:
- Automatic delivery, không cần polling
- Built-in partition management và load balancing
- Automatic checkpointing
- Simpler implementation
- **Phù hợp cho:** Continuous embedding refresh production

### Pull Model

Code của bạn tự poll changes:
- Kiểm soát khi nào và cách process
- Resource consumption thấp hơn cho batch processing
- Không có external dependencies
- **Phù hợp cho:** One-time migration, batch refresh, infrequent processing

---

## Azure Functions Cosmos DB Trigger (Push Model)

Đây là cách đơn giản nhất — trigger tự động invoke function khi có change:

```python
import azure.functions as func
from openai import AzureOpenAI
from azure.cosmos import CosmosClient
import os

app = func.FunctionApp()

@app.cosmos_db_trigger(
    arg_name="documents",
    container_name="knowledge-base",
    database_name="support-db",
    connection="CosmosDBConnection",
    lease_container_name="leases",                      # Container theo dõi progress
    create_lease_container_if_not_exists=True           # Tự tạo lease container
)
def refresh_embeddings(documents: func.DocumentList):
    if not documents:
        return

    openai_client = AzureOpenAI(
        api_key=os.environ["OPENAI_API_KEY"],
        api_version="2024-02-01",
        azure_endpoint=os.environ["OPENAI_ENDPOINT"]
    )
    container = CosmosClient.from_connection_string(
        os.environ["CosmosDBConnection"]
    ).get_database_client("support-db").get_container_client("knowledge-base")

    for doc in documents:
        if should_refresh_embedding(doc):
            text_content = f"{doc.get('title', '')} {doc.get('content', '')}"

            response = openai_client.embeddings.create(
                input=text_content,
                model="text-embedding-ada-002"
            )

            doc["embedding"] = response.data[0].embedding
            container.upsert_item(doc)
```

---

## Lease Container

Change feed processor cần **separate lease container** để track progress:

- **Checkpointing** — nếu processing fail, resume từ checkpoint, không reprocess all
- **Coordination** — nhiều processor instance chia partitions giữa nhau
- **Failover** — nếu một instance fail, instance khác acquire leases

**Yêu cầu:** 400 RU/s thường đủ. Tên khác với data container. Multiple functions monitor cùng container → cùng lease container với khác `leaseContainerPrefix`.

---

## Selective Embedding Refresh

Regenerate embedding cho **mọi** change tốn tiền (embedding API cost + rate limits). Chỉ refresh khi content thực sự thay đổi:

```python
def should_refresh_embedding(current_doc, previous_doc=None):
    """
    Chỉ refresh nếu content ảnh hưởng đến embedding thay đổi.
    Metadata-only change (category, status...) không cần refresh.
    """
    if previous_doc is None:
        return "embedding" not in current_doc

    content_fields = ["title", "content", "description", "summary"]
    for field in content_fields:
        if current_doc.get(field, "") != previous_doc.get(field, ""):
            return True

    return False    # Chỉ metadata thay đổi → không cần refresh
```

### Dùng content hash (efficient hơn)

```python
import hashlib

def compute_content_hash(doc):
    text_content = f"{doc.get('title', '')} {doc.get('content', '')}"
    return hashlib.sha256(text_content.encode()).hexdigest()

def should_refresh_embedding(doc):
    current_hash = compute_content_hash(doc)
    stored_hash = doc.get("contentHash", "")
    return current_hash != stored_hash
```

Lưu `contentHash` vào document, update khi regenerate embedding — efficient comparison không cần fetch previous version.

---

## Pull Model với Python SDK

```python
change_feed_iterator = container.query_items_change_feed(
    start_time="Beginning"    # Hoặc dùng stored continuation token
)

continuation_token = None

for page in change_feed_iterator.by_page():
    for change in page:
        if should_refresh_embedding(change):
            change["embedding"] = generate_embedding(change.get("content", ""))
            change["contentHash"] = compute_content_hash(change)
            container.upsert_item(change)

    continuation_token = change_feed_iterator.continuation_token

# Lưu continuation_token để lần sau resume
```

**Continuation token** lưu vị trí trong change feed — lưu persistent để không reprocess hay miss changes.

---

## High-Volume Handling

### Batch API calls

```python
# Thay vì gọi embedding API từng document một
texts_to_embed = [doc.get("content", "") for doc in batch_documents]

response = openai_client.embeddings.create(
    input=texts_to_embed,          # Array of texts — một API call
    model="text-embedding-ada-002"
)

for i, doc in enumerate(batch_documents):
    doc["embedding"] = response.data[i].embedding
```

### Error handling và idempotency

```python
from azure.cosmos.exceptions import CosmosResourceNotFoundError

def process_document_change(doc, container, openai_client):
    try:
        if should_refresh_embedding(doc):
            response = openai_client.embeddings.create(
                input=f"{doc.get('title', '')} {doc.get('content', '')}",
                model="text-embedding-ada-002"
            )
            doc["embedding"] = response.data[0].embedding
            doc["contentHash"] = compute_content_hash(doc)
            container.upsert_item(doc)

    except CosmosResourceNotFoundError:
        pass    # Document deleted, skip

    except Exception as e:
        logging.error(f"Failed: {doc.get('id')}: {e}")
        raise   # Retry
```

**Idempotency:** Regenerate embedding từ cùng content → cùng kết quả. Applying change 2 lần = safe.

---

## Bản chất bài này là gì?

**Một câu:** Change feed là CDC (Change Data Capture) native trong Cosmos DB — tự động detect document changes để trigger embedding refresh, tránh stale embeddings mà không cần polling.

### Polling vs Change Feed vs Event Bus

| Approach | Cơ chế | Latency | Complexity | Dùng khi |
|---|---|---|---|---|
| **Scheduled polling** | Cron job query `_ts > last_run` | Phụ thuộc interval | Thấp | Simple, ít thay đổi |
| **Change feed (push)** | Azure Functions trigger | Real-time | Trung bình | Continuous production refresh |
| **Change feed (pull)** | SDK `query_items_change_feed` | Manual poll | Trung bình | Batch, migration |
| **Event Bus (Service Bus)** | App gửi event khi write | Real-time | Cao nhất | Multi-service fan-out |

**Change feed chỉ capture insert + update, không phải delete:** Đây là limitation quan trọng. Nếu xoá document → embedding trong index không bị dọn. Phải dùng soft-delete (`isDeleted: true`) để change feed detect.

**Lease container là bắt buộc cho push model:** Không phải optional — nó lưu checkpoint để resume sau khi fail, và coordinate nhiều worker không process cùng 1 change.

**Selective refresh = cost optimization quan trọng:** Mỗi lần gọi embedding API tốn tiền + có rate limit. Chỉ refresh khi content fields (title, content) thay đổi, không phải khi metadata (status, category) thay đổi. Content hash là cách efficient nhất.

---

## Checklist ghi nhớ cho AI-200

- [ ] Change feed = **persistent ordered log** của insert và update (không có delete)
- [ ] Enabled **mặc định** trên tất cả container
- [ ] **Push model** (Azure Functions trigger) = recommended cho continuous production refresh
- [ ] **Pull model** (SDK `query_items_change_feed`) = batch processing, migration
- [ ] Azure Functions Cosmos DB trigger cần **lease container** (400 RU/s đủ)
- [ ] Lease container: checkpointing, coordination, failover
- [ ] **Selective refresh** — chỉ regenerate khi content fields thay đổi, không phải mọi update
- [ ] Content hash = efficient way to detect content change
- [ ] Batch embedding API calls để giảm cost và rate limiting
- [ ] Continuation token để resume pull model giữa các lần chạy
- [ ] Processing phải **idempotent** — apply change 2 lần = same result

---

*Module Assessment tiếp theo*

---

[← Bài 3](./cosmosdb-m2-bai3-hybrid-search.md) · [🏠 Mục lục](../README.md) · [Assessment →](./cosmosdb-m2-module-assessment.md)
