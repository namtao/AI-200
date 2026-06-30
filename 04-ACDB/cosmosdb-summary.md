# Tổng hợp kiến thức — 04-ACDB (3 Modules)

> AI-200 · Module 1: Build queries for Azure Cosmos DB for NoSQL · Module 2: Implement vector search · Module 3: Optimize query performance

---

## Bức tranh tổng thể

```
Cosmos DB for NoSQL trong AI-200:
Module 1: Core database     → Account/Container/Item, RU, SDK, SQL dialect
Module 2: Vector store      → Embedding policy, VectorDistance, Hybrid search (RRF), Change feed
Module 3: Performance       → Index types, Composite indexes, Vector index tuning, Consistency levels
```

**Tại sao Cosmos DB cho AI?**
- Unified model: vector + metadata trong cùng document → không cần separate vector DB
- Global distribution + millisecond latency
- Schema-agnostic: data model thay đổi không cần migration
- Change feed: built-in CDC để keep embeddings in sync

---

## MODULE 1 — Core: Queries, SDK, Data Model

### Resource Hierarchy

```
Account (DNS endpoint: https://<name>.documents.azure.com:443/)
└── Database (logical namespace, shared throughput scope)
    └── Container (unit of scalability, PARTITION KEY được khai báo ở đây)
        └── Item (JSON document, yêu cầu "id" property)
```

**Partition key = quyết định không thể thay đổi sau khi tạo container.**

| Partition key tốt | Partition key tệ |
|---|---|
| `userId` — high cardinality, match "lấy data của user X" | `isActive` (boolean) — chỉ 2 partition → hot |
| `categoryId` — query thường theo category | timestamp — hot partition trong peak period |
| `modelId` — group inference cache | status với 3-4 values — low cardinality |

### RU Model

| Operation | RU cost |
|---|---|
| Point read (id + partition key) | ~1 RU |
| Write (1KB item) | ~5-10 RU |
| Single-partition query | ~3-5 RU |
| Cross-partition query | Nhiều lần hơn — tỷ lệ với số partitions |
| Strong/Bounded staleness READ | 2× so với Session/Eventual |

**Throughput modes:**
- Manual: fix số RU/s, min **400 RU/s**
- Autoscale: 10%–100% của max, min autoscale max **1000 RU/s**

### System Properties

| Property | Ý nghĩa |
|---|---|
| `_etag` | Optimistic concurrency — thay đổi mỗi khi item update |
| `_ts` | Unix timestamp của last modification |
| `_rid` | Internal resource ID |

### SDK Patterns

```python
# Khởi tạo — tạo MỘT LẦN, dùng suốt app lifecycle
from azure.cosmos import CosmosClient
from azure.identity import DefaultAzureCredential

client = CosmosClient(endpoint, credential=DefaultAzureCredential())
database = client.get_database_client("aidata")
container = database.get_container_client("documents")
```

**Auth:**
- Dev: Account key hoặc Azure CLI (`az login`)
- Production: `DefaultAzureCredential()` → Managed Identity
- RBAC roles: `Cosmos DB Built-in Data Reader` · `Cosmos DB Built-in Data Contributor`

**CRUD Methods:**

| Method | Behavior | Dùng khi |
|---|---|---|
| `create_item()` | Fail 409 nếu đã tồn tại | Cần detect duplicate |
| `upsert_item()` | Insert hoặc replace | Cache, sync, mặc định tốt |
| `replace_item()` | Fail nếu không tồn tại | Cần optimistic concurrency với `_etag` |
| `read_item(item, partition_key)` | ~1 RU, point read | Fetch khi biết id + partition key |
| `delete_item()` | Xóa | Cleanup |

**Optimistic concurrency:**
```python
container.replace_item(item=item["id"], body=item, if_match=item["_etag"])
# Fail với CosmosAccessConditionFailedError nếu item đã thay đổi bởi process khác
```

### Query

```python
# Point read — prefer khi biết id + partition key
item = container.read_item(item="doc-123", partition_key="networking")

# Single-partition query (rẻ hơn)
items = container.query_items(
    query="SELECT p.id, p.name FROM products p WHERE p.categoryId = @cat AND p.price < @max",
    parameters=[{"name": "@cat", "value": "electronics"}, {"name": "@max", "value": 500}],
    partition_key="electronics"
)

# Cross-partition (khi không có partition key trong filter)
items = container.query_items(query=query, parameters=params, enable_cross_partition_query=True)
```

**Luôn dùng parameterized query:** Security (injection prevention) + performance (query plan cache).

**VALUE keyword:** Trả về scalar/array trực tiếp — `SELECT VALUE p.name` → `["Speaker", "Router"]`

---

## MODULE 2 — Vector Search + Change Feed

### Setup Vector Container

```bash
# Bước 1: Enable vector search (mất ~15 phút, một lần per account)
az cosmosdb update --capabilities EnableNoSQLVectorSearch \
    --name <account> --resource-group <rg>
```

```python
# Bước 2: Vector embedding policy — BẮT BUỘC, KHÔNG THỂ THAY ĐỔI SAU KHI TẠO
vector_embedding_policy = {
    "vectorEmbeddings": [{
        "path": "/embedding",
        "dataType": "float32",          # float32 (full) | float16 (50% storage)
        "distanceFunction": "cosine",   # cosine (recommended text) | dotproduct | euclidean
        "dimensions": 1536              # PHẢI match embedding model
    }]
}

# Bước 3: Indexing policy — exclude embedding khỏi range index
indexing_policy = {
    "includedPaths": [{"path": "/*"}],
    "excludedPaths": [
        {"path": "/\"_etag\"/?"},
        {"path": "/embedding/*"}        # QUAN TRỌNG: exclude khỏi range index
    ],
    "vectorIndexes": [
        {"path": "/embedding", "type": "diskANN"}  # Vector index riêng
    ]
}

container = database.create_container(
    id="knowledge-base",
    partition_key=PartitionKey(path="/category"),
    indexing_policy=indexing_policy,
    vector_embedding_policy=vector_embedding_policy
)
```

### Vector Index Types

| Index | Max dim | Dataset | Accuracy | Dùng khi |
|---|---|---|---|---|
| `flat` | **505** | Nhỏ | 100% exact | Small dataset, exact required |
| `quantizedFlat` | 4,096 | ~1K–50K/partition | ~Exact | Medium |
| `diskANN` | 4,096 | >50K/partition | High (approx) | **Large production** ✅ |

`quantizedFlat` và `diskANN` cần **ít nhất 1,000 vectors** để effective.

### Embedding Models

| Model | Dimensions | Ghi chú |
|---|---|---|
| `text-embedding-ada-002` | **1,536** | Phổ biến, normalized, cosine recommended |
| `text-embedding-3-large` | 3,072 | Chất lượng cao hơn, storage lớn hơn |

### Vector Search Queries

```python
# Bước 1: Embed query text bằng CÙNG MODEL với documents
response = openai_client.embeddings.create(input=query_text, model="text-embedding-ada-002")
query_embedding = response.data[0].embedding

# Bước 2: Vector search — ORDER BY và TOP N là BẮT BUỘC
query = """
    SELECT TOP 10
        c.id, c.title, c.category,
        VectorDistance(c.embedding, @queryVector) AS SimilarityScore
    FROM c
    ORDER BY VectorDistance(c.embedding, @queryVector)
"""
results = container.query_items(
    query=query,
    parameters=[{"name": "@queryVector", "value": query_embedding}],
    enable_cross_partition_query=True
)
```

**Cosine similarity thresholds:**
- 0.7+ = Highly similar
- 0.5–0.7 = Moderately similar
- < 0.5 = Low similarity

**Filter theo threshold + single-partition optimization:**
```sql
SELECT TOP 10 c.id, c.title, VectorDistance(c.embedding, @queryVector) AS Score
FROM c
WHERE c.category = @category
  AND VectorDistance(c.embedding, @queryVector) > 0.7
ORDER BY VectorDistance(c.embedding, @queryVector)
```

```python
results = container.query_items(query=query, parameters=[...], partition_key="networking")
```

**Brute-force (testing only):**
```sql
VectorDistance(c.embedding, @queryVector, true)   -- third param = true
```

### Hybrid Search (RRF)

```sql
SELECT TOP 10 *
FROM c
ORDER BY RANK RRF(
    VectorDistance(c.embedding, @queryVector),          -- Semantic score
    FullTextScore(c.content, @searchTerm1, @term2),     -- Keyword score
    [2, 1]                                              -- Vector weight 2×, text weight 1×
)
```

Cần config khi tạo container: vector policy + full-text policy + fullTextIndexes.

**Pre-filter vs Post-filter:** Query optimizer tự quyết. Filter selective (loại nhiều document) → pre-filter → tiết kiệm RU. Đảm bảo filter properties có index.

### Change Feed — Embedding Sync

```python
# Push model — Azure Functions trigger (production recommended)
@app.cosmos_db_trigger(
    arg_name="documents",
    container_name="knowledge-base",
    database_name="support-db",
    connection="CosmosDBConnection",
    lease_container_name="leases",
    create_lease_container_if_not_exists=True
)
def refresh_embeddings(documents: func.DocumentList):
    for doc in documents:
        if should_refresh_embedding(doc):
            doc["embedding"] = generate_embedding(doc["content"])
            container.upsert_item(doc)
```

**Change feed đặc tính:**
- Enabled mặc định trên tất cả containers
- Chỉ capture **insert và update** — **KHÔNG** capture delete
- Ordered per partition key
- Push model (Azure Functions) = automatic, checkpointed
- Pull model (SDK `query_items_change_feed`) = batch, migration

**Selective refresh = cost savings:**
```python
def should_refresh_embedding(doc):
    current_hash = hashlib.sha256(f"{doc.get('title')}{doc.get('content')}".encode()).hexdigest()
    return current_hash != doc.get("contentHash", "")
```

---

## MODULE 3 — Index Optimization + Consistency

### Index Types

| Index | Hỗ trợ | Dùng khi |
|---|---|---|
| **Range** | `=`, `>`, `<`, `ORDER BY` single, string functions | Filter theo type, status, date |
| **Composite** | Multi-property `ORDER BY`, filter + sort | Sort 2+ fields, filter + sort |
| **Vector** | `VectorDistance()` | Semantic search |
| **Spatial** | `ST_DISTANCE`, `ST_WITHIN` | Geo queries |
| **Full-text** | `FullTextScore()` | Keyword search for RRF |
| **Tuple** | Array element multi-property filter | Chunked documents |

**Indexing Modes:**
- `consistent` (default): sync update khi write
- `none`: chỉ point read, không query
- Lazy: **DEPRECATED**

### Selective Indexing Policy

```json
{
  "indexingMode": "consistent",
  "includedPaths": [
    { "path": "/category/?" },
    { "path": "/documentType/?" },
    { "path": "/uploadDate/?" }
  ],
  "excludedPaths": [
    { "path": "/*" },
    { "path": "/embedding/*" }
  ],
  "vectorIndexes": [
    { "path": "/embedding", "type": "diskANN" }
  ]
}
```

> Khi dùng selective (exclude `/*`): partition key path phải được **explicitly include**.

### Composite Indexes

**ORDER BY 2+ properties → BẮT BUỘC composite index — thiếu thì query fail (không chỉ slow).**

```json
{
  "compositeIndexes": [[
    { "path": "/documentType", "order": "ascending" },   // equality first
    { "path": "/uploadDate", "order": "descending" }     // range/sort last
  ]]
}
```

**Rules:**
- Equality filters → đặt trước
- Range filter → đặt cuối, tối đa 1 range filter per composite index
- `(A DESC, B DESC)` cũng support `(A ASC, B ASC)` — KHÔNG support mixed: `(A ASC, B DESC)`

**Transform behavior:**
- Thêm index → async transform → query chưa dùng ngay
- **Xóa index → query NGAY LẬP TỨC ngừng dùng** → fall back to scan
- Best practice: add new → chờ transform xong → remove old

### Query Metrics

```python
results = container.query_items(
    query=query, parameters=parameters,
    populate_query_metrics=True,
    response_hook=lambda headers, _: print(headers.get("x-ms-documentdb-query-metrics"))
)
```

**Low index utilization + High retrieved/output ratio = MISSING INDEX.**

### 5 Consistency Levels

```
Strong ←————————— Session ————————————→ Eventual
2× RU read                              1× RU read
```

| Level | RU Read Cost | Guarantee | Dùng khi |
|---|---|---|---|
| **Strong** | **2×** | Always latest | Financial, compliance |
| **Bounded staleness** | **2×** | Lag ≤ K versions | Near-strong |
| **Session** ⭐ | 1× | Read-your-writes in session | **AI app user session** |
| **Consistent prefix** | 1× | No out-of-order | Event streams |
| **Eventual** | 1× | Highest availability | Analytics, background |

**Session consistency cho AI app:**
```python
client = CosmosClient(url=endpoint, credential=credential,
                      consistency_level=ConsistencyLevel.Session)

# Capture session token từ write
session_info = {}
container.create_item(body=doc,
    response_hook=lambda h, _: session_info.update({"token": h.get("x-ms-session-token")}))

# Read với session token → guaranteed read-your-writes
results = container.query_items(query=q, parameters=p,
                                session_token=session_info.get("token"))
```

Session token là **partition-bound** — distributed app cần pass explicitly.

---

## CLI Quick Reference

```bash
# === Account ===
az cosmosdb create --name myaccount --resource-group rg --kind GlobalDocumentDB
az cosmosdb update --capabilities EnableNoSQLVectorSearch --name myaccount --resource-group rg

# === Database ===
az cosmosdb sql database create --account-name myaccount --resource-group rg --name mydb

# === Container ===
az cosmosdb sql container create \
    --account-name myaccount --resource-group rg \
    --database-name mydb --name mycontainer \
    --partition-key-path "/categoryId" \
    --throughput 400

# === Connection string ===
az cosmosdb keys list --name myaccount --resource-group rg --type connection-strings
```

**Python SDK quick setup:**
```python
from azure.cosmos import CosmosClient, PartitionKey, ThroughputProperties
from azure.identity import DefaultAzureCredential

client = CosmosClient(endpoint, DefaultAzureCredential())
db = client.create_database_if_not_exists("mydb")
container = db.create_container_if_not_exists(
    id="mycontainer",
    partition_key=PartitionKey(path="/categoryId"),
    offer_throughput=400
)
```

---

## Exam Traps — Những điểm hay nhầm

1. **Partition key không thể thay đổi sau khi tạo container** — phải migrate data sang container mới.

2. **Vector policy không thể thay đổi sau khi tạo container** — đổi embedding model (1536→3072) = phải tạo container mới.

3. **Exclude embedding khỏi range index là bắt buộc** — không exclude → storage phình, write chậm, không có ích gì vì vector search dùng vector index, không phải range index.

4. **`quantizedFlat` và `diskANN` cần ít nhất 1,000 vectors** — dưới ngưỡng → brute-force scan, có thể tốn RU hơn dự kiến.

5. **ORDER BY và TOP N là bắt buộc trong vector search** — thiếu TOP N → query return tất cả documents → RU rất lớn. Thiếu ORDER BY → kết quả không được rank theo relevance.

6. **Query embedding phải dùng cùng model với document embedding** — khác model = incompatible vector space = kết quả vô nghĩa.

7. **Change feed KHÔNG capture delete** — dùng soft-delete (`isDeleted: true`) nếu cần track delete.

8. **Composite index bắt buộc cho ORDER BY 2+ properties** — thiếu → query **fail** (không chỉ slow). Xóa composite index → query fail ngay lập tức, không chờ gì.

9. **Mixed direction không support trong 1 composite index** — `(A ASC, B DESC)` cần composite index riêng.

10. **Partition key phải explicitly include khi dùng selective indexing** — exclude `/*` → partition key path cũng bị exclude nếu không explicit include.

11. **Strong/Bounded staleness = 2× RU read cost** — Session consistency đủ cho hầu hết AI app và tiết kiệm 50% read cost.

12. **Session token là partition-bound** — distributed service cần pass token explicitly để guarantee read-your-writes.

13. **`create_item()` fail với 409 nếu đã tồn tại; `replace_item()` fail nếu không tồn tại** — dùng `upsert_item()` khi không cần phân biệt.

14. **Cross-partition query = đắt** — luôn include partition key trong query hoặc `partition_key=` parameter khi có thể.

---

## Assessment Summary

### Module 1 — Build Queries
| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Partition key cho user interaction logs | `userId` (high cardinality, match query) |
| 2 | Cache inference result — insert hoặc replace | `upsert_item()` |
| 3 | Retrieve item khi biết id + partition key | `read_item()` (point read ~1 RU) |
| 4 | Lý do dùng parameterized query | Security + query plan caching |
| 5 | Giảm RU cho price range query | Add partition key vào WHERE clause |

### Module 2 — Vector Search
| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Vector policy cho `text-embedding-ada-002` | float32, 1536 dimensions, cosine |
| 2 | Giảm RU vector search | Reduce TOP N (10-20 thay vì 100) |
| 3 | Vector search + category filter efficiently | WHERE clause + partition_key parameter |
| 4 | Hybrid semantic + exact keyword search | ORDER BY RANK RRF |
| 5 | Auto embedding refresh khi document update | Azure Functions Cosmos DB trigger |

### Module 3 — Optimize Performance
| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Composite index cho filter + sort | (documentType ASC, uploadDate DESC) |
| 2 | Vector index cho 500K vectors/partition | diskANN |
| 3 | Giảm storage từ embedding arrays | Exclude range index + add vector index |
| 4 | User không thấy document vừa upload | Session consistency + session tokens |
| 5 | Low index utilization + high retrieved/output | Thêm range/composite index cho missing |

---

## Cosmos DB vs Relational vs Dedicated Vector DB

| Tính năng | SQL Server / PostgreSQL | Dedicated Vector DB (Pinecone) | Cosmos DB NoSQL |
|---|---|---|---|
| Data model | Tables + Rows | Vectors + metadata | JSON documents |
| Schema | Rigid | Semi-flexible | Schema-agnostic |
| Vector search | Extension (pgvector) | Native + optimized | Native (diskANN) |
| Metadata filter | SQL WHERE | Filter conditions | SQL WHERE |
| Global distribution | Manual/limited | Limited | ✅ Built-in |
| Change feed | CDC tools | ❌ | ✅ Built-in |
| Single system | ❌ (need pgvector) | ❌ (need separate DB) | ✅ |
| Operational complexity | Thấp | Cao (2 systems) | Thấp |

---

[← Assessment](./cosmosdb-m3-module-assessment.md) · [🏠 Mục lục](../README.md) · [Mục lục →](../README.md)
