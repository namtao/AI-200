# Bài 3 — Tune Vector Indexes + Reduce RU Costs + Consistency Levels

> Khoá: AI-200 · Cosmos DB — Optimize query performance for Azure Cosmos DB for NoSQL

---

## Vector Indexes trong Indexing Policy

Vector index hoạt động song song với range và composite indexes. Cần khai báo **vector policy** trước khi add vector index.

```json
{
  "indexingMode": "consistent",
  "includedPaths": [
    { "path": "/category/?" },
    { "path": "/createdDate/?" }
  ],
  "excludedPaths": [
    { "path": "/*" },
    { "path": "/embedding/*" }    // Exclude khỏi range index
  ],
  "compositeIndexes": [
    [
      { "path": "/category", "order": "ascending" },
      { "path": "/createdDate", "order": "descending" }
    ]
  ],
  "vectorIndexes": [
    { "path": "/embedding", "type": "diskANN" }
  ]
}
```

---

## Chọn Vector Index Type

| Index | Dataset size | Max dim | Accuracy | Khi dùng |
|---|---|---|---|---|
| **flat** | Nhỏ | **505** | 100% exact | Small dataset, metadata pre-filter reduces set dramatically |
| **quantizedFlat** | ~1K–50K vectors/partition | 4,096 | ~Exact (slight reduction) | Medium dataset, slight accuracy trade-off OK |
| **diskANN** | >50K vectors/partition | 4,096 | High (approximate) | **Large production dataset** — recommended |

**Cả quantizedFlat và diskANN cần ít nhất 1,000 vectors** để effective. Dưới 1,000 → brute-force scan.

### Optional tuning parameters

```json
{
  "vectorIndexes": [
    {
      "path": "/embedding",
      "type": "diskANN",
      "quantizationByteSize": 64,      // 1-512: lớn hơn = accuracy tốt hơn, storage nhiều hơn
      "indexingSearchListSize": 150     // diskANN only, 10-500, default 100: cao hơn = index chất lượng tốt hơn
    }
  ]
}
```

Dùng default parameters trừ khi có accuracy issue hoặc specific latency targets.

---

## Strategic Indexing — Giảm RU

### Phân tích query patterns trước khi design index

Catalog:
- **Filter properties** — WHERE clause, equality hay range?
- **Sort properties** — ORDER BY, hướng nào?
- **Property combinations** — fields nào xuất hiện cùng nhau?
- **Access frequency** — query nào chạy thường xuyên nhất?

### Dùng Query Metrics

```python
query_metrics_output = {}

def response_hook(headers, results):
    query_metrics_output["metrics"] = headers.get("x-ms-documentdb-query-metrics", "")
    query_metrics_output["charge"] = headers.get("x-ms-request-charge", "")

results = container.query_items(
    query=query,
    parameters=parameters,
    populate_query_metrics=True,
    response_hook=response_hook
)
items = list(results)
print(f"RU: {query_metrics_output.get('charge', '')} | Metrics: {query_metrics_output.get('metrics', '')}")
```

**Key metrics:**
- **Index utilization %:** Thấp → cần index
- **Retrieved document count** cao so với **Output document count** → inefficient filtering → cần index

### Exclude properties không được query

```json
// Selective policy — chỉ index properties trong WHERE/ORDER BY
{
  "includedPaths": [
    { "path": "/title/?" },
    { "path": "/documentType/?" },
    { "path": "/category/?" },
    { "path": "/uploadDate/?" }
  ],
  "excludedPaths": [{ "path": "/*" }],    // content, summary, embedding không index
  "vectorIndexes": [{ "path": "/embedding", "type": "diskANN" }]
}
```

Large content fields, summary text, embedding arrays chỉ được read (display) → không cần index.

### Write-heavy vs Read-heavy

| Profile | Strategy |
|---|---|
| **Write-heavy** (ingestion, streaming) | Minimize indexes — ít overhead per write |
| **Read-heavy** (search, retrieval — hầu hết AI app) | Comprehensive indexing — reduce query cost across many reads |

---

## 5 Consistency Levels

| Level | Guarantee | Replicas read | RU read cost | Dùng khi |
|---|---|---|---|---|
| **Strong** | Linearizability — always latest | 2 (minority quorum) | **2×** | Financial, regulatory compliance |
| **Bounded staleness** | Lag ≤ K versions hoặc T seconds | 2 | **2×** | Near-strong, single write region |
| **Session** | Read-your-writes trong session | 1 | 1× | **Most AI apps** — user thấy upload của mình |
| **Consistent prefix** | No out-of-order reads | 1 | 1× | Ordered sequences (event streams) |
| **Eventual** | Highest availability, weakest guarantee | 1 | 1× | Analytics, background processing |

### Session Consistency cho AI Application

```python
from azure.cosmos import CosmosClient, ConsistencyLevel

client = CosmosClient(url=endpoint, credential=credential,
                      consistency_level=ConsistencyLevel.Session)
container = client.get_database_client("db").get_container_client("container")

# Capture session token từ write
session_info = {}
def capture_token(headers, result):
    session_info["token"] = headers.get("x-ms-session-token", "")

container.create_item(body=document, response_hook=capture_token)

# Pass session token vào subsequent read → guaranteed read-your-writes
results = container.query_items(
    query="SELECT * FROM c WHERE c.category = @cat",
    parameters=[{"name": "@cat", "value": "proposals"}],
    session_token=session_info.get("token")
)
```

**Session token là partition-bound.** SDK quản lý tự động trong cùng client instance. Distributed app với nhiều service → pass token explicitly.

### Consistency và RU Cost

```
Strong / Bounded staleness → 2 replicas → 2× RU read cost
Session / Consistent prefix / Eventual → 1 replica → 1× RU read cost
```

→ Dùng Session thay Strong = **giảm 50% read RU** với practical consistency cho user-facing ops.

### Override per request

```python
# Strong cho critical reads
strong_client = CosmosClient(url=endpoint, credential=credential,
                             consistency_level=ConsistencyLevel.Strong)

# Eventual cho analytics
eventual_client = CosmosClient(url=endpoint, credential=credential,
                               consistency_level=ConsistencyLevel.Eventual)
```

Tạo nhiều client instance với consistency level khác nhau cho các loại operation khác nhau.

---

## Bản chất bài này là gì?

**Một câu:** Session consistency là sweet spot cho AI app (read-your-writes + 1× RU cost) — Strong và Bounded staleness tốn 2× RU read cost mà hầu hết AI workload không cần strict linearizability đó.

### 5 Consistency Levels — Mental Model

```
Strong ←——————————————————————————————→ Eventual
Strongest                            Weakest
Slowest                               Fastest
2× RU                                  1× RU

Strong  >  Bounded staleness  >  Session  >  Consistent prefix  >  Eventual
```

| Level | Tương đương trong SQL | Dùng khi |
|---|---|---|
| Strong | Serializable + sync replication | Financial transactions, regulatory |
| Bounded staleness | Near-serializable, known lag | Single write region production |
| **Session** | Read-your-writes | **AI app user session** ✅ |
| Consistent prefix | No disorder | Event stream ordering |
| Eventual | Async replication | Analytics, batch processing |

**Session vs Strong — 50% cost savings:** Strong = 2 replicas → 2× RU. Session = 1 replica → 1× RU. Nếu AI app xử lý 1 triệu reads/ngày với Session thay Strong → tiết kiệm 50% read RU cost.

**Session token là partition-bound:** Khi nhiều service instances cùng viết/đọc (distributed AI backend), phải pass session token explicitly. Nếu không pass → có thể đọc outdated data dù vừa write.

---

## Checklist ghi nhớ cho AI-200

- [ ] Vector index selection: **flat** (≤505 dim, small) · **quantizedFlat** (≤50K) · **diskANN** (large, production)
- [ ] `quantizedFlat` và `diskANN` cần **1,000+ vectors** để effective
- [ ] `quantizationByteSize` (1-512) · `indexingSearchListSize` (10-500, diskANN only)
- [ ] Query metrics: low index utilization + high retrieved/output ratio = **missing index**
- [ ] `populate_query_metrics=True` để lấy metrics từ response headers
- [ ] Selective indexing: exclude `/*` → include chỉ queried properties
- [ ] Write-heavy → ít indexes · Read-heavy → comprehensive indexes
- [ ] 5 levels: Strong > Bounded staleness > **Session** > Consistent prefix > Eventual
- [ ] Strong và Bounded staleness = **2×** RU read cost
- [ ] **Session** = recommended cho AI app — read-your-writes, 1× RU
- [ ] Session token là partition-bound, cần pass explicitly trong distributed app
- [ ] Eventual = analytics, background — không cần fresh data

---

*Module Assessment tiếp theo*

---

[← Bài 2](./cosmosdb-m3-bai2-range-composite-indexes.md) · [🏠 Mục lục](../README.md) · [Assessment →](./cosmosdb-m3-module-assessment.md)
