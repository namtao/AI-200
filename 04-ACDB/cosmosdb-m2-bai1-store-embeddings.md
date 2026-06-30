# Bài 1 — Store and Retrieve Embeddings in Azure Cosmos DB

> Khoá: AI-200 · Cosmos DB — Implement vector search on Azure Cosmos DB for NoSQL

---

## Vector Embeddings là gì?

Embedding là **numerical representation của data** do ML model generate. Text, image, hay content khác được chuyển thành array of floating-point numbers — "vector" — capture semantic meaning.

**Khi content có nghĩa gần nhau → vector gần nhau trong vector space.** Hai document về WiFi troubleshooting có vector gần nhau dù một cái dùng "wireless connection" và cái kia dùng "WiFi network" — semantic search tìm được cả hai khi search "internet not working".

**Embedding models và dimensions:**

| Model | Dimensions | Ghi chú |
|---|---|---|
| `text-embedding-ada-002` (Azure OpenAI) | 1,536 | Phổ biến, normalized vectors |
| `text-embedding-3-large` (Azure OpenAI) | 3,072 | Chất lượng cao hơn, tốn storage hơn |

Dimension nhiều hơn = capture được nhiều nuance hơn, nhưng storage và compute cost cao hơn.

---

## Document Structure cho Vector Search

Lưu embedding như một **array property** bên cạnh business data:

```json
{
    "id": "doc-12345",
    "title": "Troubleshooting wireless network connections",
    "content": "This guide covers common WiFi connectivity issues...",
    "category": "networking",
    "productId": "router-x100",
    "createdDate": "2024-06-15T10:30:00Z",
    "embedding": [0.023, -0.041, 0.067, ...]   // 1,536 floats
}
```

Lợi ích unified data model: vector + metadata trong cùng database → không cần separate vector database → đơn giản hóa architecture.

---

## Bật Vector Search Feature

Phải enable trước khi tạo container (auto-approved, mất ~15 phút):

```bash
az cosmosdb update --capabilities EnableNoSQLVectorSearch \
    --name <account-name> --resource-group <rg>
```

Hoặc qua Portal: Settings → Features → Vector Search for NoSQL API.

---

## Vector Embedding Policy

Khi tạo container, **bắt buộc** khai báo vector policy. **Không thể thay đổi sau khi tạo.**

```python
vector_embedding_policy = {
    "vectorEmbeddings": [
        {
            "path": "/embedding",          # JSON path đến embedding property
            "dataType": "float32",         # Kiểu dữ liệu của vector elements
            "distanceFunction": "cosine",  # Metric tính similarity
            "dimensions": 1536             # Phải match với embedding model
        }
    ]
}
```

### Distance Functions

| Function | Tính gì | Score range | Dùng khi |
|---|---|---|---|
| **cosine** | Góc giữa 2 vector | -1 đến +1 (thực tế 0-1) | Text embeddings từ Azure OpenAI — recommended |
| **dotproduct** | Góc + magnitude | Varies | Normalized vectors, nhanh hơn cosine một chút |
| **euclidean** | Khoảng cách thẳng | 0 (identical) đến ∞ | Specialized cases, ít dùng với text |

Azure OpenAI embeddings là **normalized** → cosine và dotproduct cho kết quả giống nhau về mặt toán học, cosine được recommend vì intuitive hơn.

### Data Types — Trade-off precision vs storage

| Data type | Storage | Accuracy | Dùng khi |
|---|---|---|---|
| `float32` | Full | Full | Bắt đầu ở đây, default |
| `float16` | 50% của float32 | Gần như tương đương | Khi storage cost là vấn đề |
| `int8` / `uint8` | Compressed | Lower (quantization) | Large-scale, cần đánh giá kỹ |

---

## Vector Index Types

| Index | Max dimensions | Search type | Dùng khi |
|---|---|---|---|
| `flat` | **505** | Exact brute-force | Small dataset, exact results required |
| `quantizedFlat` | 4,096 | Compressed brute-force | Đến ~50,000 vectors per partition |
| `diskANN` | 4,096 | Fast approximate | Large dataset, millions of vectors — recommended |

> **Lưu ý:** `quantizedFlat` và `diskANN` cần **ít nhất 1,000 vectors** mới effective. Dưới 1,000 → fallback to full scan → có thể tốn RU hơn.

---

## Indexing Policy

**Quan trọng:** Exclude embedding path khỏi standard range index:

```python
indexing_policy = {
    "indexingMode": "consistent",
    "automatic": True,
    "includedPaths": [
        {"path": "/*"}
    ],
    "excludedPaths": [
        {"path": "/\"_etag\"/?"},
        {"path": "/embedding/*"}   # Exclude embedding khỏi range index
    ],
    "vectorIndexes": [
        {"path": "/embedding", "type": "diskANN"}   # Vector index riêng
    ]
}
```

**Tại sao exclude embedding?** Embedding array không benefit từ standard range index. Include nó vào sẽ tăng write cost và storage mà không có ích gì.

---

## Tạo Container với Vector Support

```python
from azure.cosmos import CosmosClient, PartitionKey

container = database.create_container(
    id="knowledge-base",
    partition_key=PartitionKey(path="/category"),
    indexing_policy=indexing_policy,
    vector_embedding_policy=vector_embedding_policy
)
```

Dùng `create_container()` (không phải `_if_not_exists`) vì vector policy không thể modify sau khi tạo → cần kiểm soát chặt.

---

## Insert Document với Embedding

```python
from openai import AzureOpenAI
from azure.cosmos import CosmosClient

openai_client = AzureOpenAI(api_key=api_key, api_version="2024-02-01", azure_endpoint=endpoint)

# Generate embedding từ document content
document_text = f"{title} {content}"
response = openai_client.embeddings.create(
    input=document_text,
    model="text-embedding-ada-002"
)
embedding = response.data[0].embedding

# Tạo document với embedding
document = {
    "id": document_id,
    "title": title,
    "content": content,
    "category": category,
    "productId": product_id,
    "embedding": embedding    # Array of 1,536 floats
}

container.upsert_item(document)    # Upsert để handle cả create và update
```

**Dùng `upsert_item()`**: khi content thay đổi, generate lại embedding và upsert — giữ embedding sync với content.

---

## Multiple Embeddings Per Document

```python
vector_embedding_policy = {
    "vectorEmbeddings": [
        {
            "path": "/titleEmbedding",
            "dataType": "float32",
            "distanceFunction": "cosine",
            "dimensions": 1536
        },
        {
            "path": "/contentEmbedding",
            "dataType": "float32",
            "distanceFunction": "cosine",
            "dimensions": 1536
        }
    ]
}
```

Dùng khi: search theo title similarity, content similarity, hoặc kết hợp cả hai.

---

## Bản chất bài này là gì?

**Một câu:** Cosmos DB unified model = lưu embedding + metadata trong cùng một document, loại bỏ nhu cầu separate vector database — đơn giản hóa architecture nhưng đòi hỏi khai báo vector policy không thể thay đổi khi tạo container.

### Cosmos DB vs Dedicated Vector DB

| | Cosmos DB (Unified) | Pinecone / Weaviate / Qdrant |
|---|---|---|
| Architecture | 1 database cho tất cả | Riêng: Cosmos DB + Vector DB |
| Metadata + vector | Cùng document | Sync riêng (2 writes, risk drift) |
| Filter khi search | WHERE + VectorDistance | Metadata filter trong query |
| Thay đổi schema | Thêm field bất kỳ lúc | Phụ thuộc vào vector DB |
| Vector policy | Fixed khi tạo container | Thay đổi được |
| Operational complexity | Thấp (1 service) | Cao (2 services, sync) |
| Max dimensions | 4,096 (diskANN) | Thường cao hơn |

**Unified model tradeoff:** Đơn giản hơn nhưng inflexible — vector policy (dimensions, distance function) không thể thay đổi sau khi tạo container. Nếu đổi embedding model (1536 → 3072 dimensions) → phải tạo container mới và migrate data.

**Exclude embedding khỏi range index là BẮT BUỘC:** 1,536 floats trong range index = lãng phí storage cực lớn, không có ích gì. Vector index là loại index riêng cho embedding.

---

## Checklist ghi nhớ cho AI-200

- [ ] Enable vector search: `az cosmosdb update --capabilities EnableNoSQLVectorSearch` (mất ~15 phút)
- [ ] Vector policy **bắt buộc** khi tạo container, **không thể thay đổi sau đó**
- [ ] `text-embedding-ada-002` → **1,536 dimensions**, `text-embedding-3-large` → 3,072
- [ ] Distance functions: **cosine** (recommended cho text), dotproduct (normalized vectors), euclidean
- [ ] Cosine score: +1 = identical, 0 = unrelated, negative = opposite
- [ ] Data types: `float32` (full precision) · `float16` (50% storage, similar quality)
- [ ] Index types: `flat` (max 505 dim) · `quantizedFlat` (~50K vectors) · `diskANN` (millions)
- [ ] `diskANN` và `quantizedFlat` cần **ít nhất 1,000 vectors** mới effective
- [ ] **Exclude embedding path** khỏi standard range index trong indexing policy
- [ ] Query embedding phải dùng **cùng model** với document embeddings

---

[← Assessment](./cosmosdb-m1-module-assessment.md) · [🏠 Mục lục](../README.md) · [Bài 2 →](./cosmosdb-m2-bai2-vector-queries.md)
