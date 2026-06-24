# Bài 2 — Implement the Azure Cosmos DB for NoSQL SDK

> Khoá: AI-200 · Cosmos DB — Build queries for Azure Cosmos DB for NoSQL

---

## Connect bằng SDK

`CosmosClient` là entry point cho tất cả interaction với Cosmos DB. Client quản lý connection pooling, request routing, và failover transparently.

### Khởi tạo với Account Key

```python
from azure.cosmos import CosmosClient

endpoint = "https://mycosmosaccount.documents.azure.com:443/"
key = "<your-account-key>"

client = CosmosClient(endpoint, credential=key)
```

### Khởi tạo với Microsoft Entra ID (DefaultAzureCredential)

```python
from azure.cosmos import CosmosClient
from azure.identity import DefaultAzureCredential

endpoint = "https://mycosmosaccount.documents.azure.com:443/"
credential = DefaultAzureCredential()

client = CosmosClient(endpoint, credential=credential)
```

---

## Authentication — Key vs Entra ID

| | Account Key | Microsoft Entra ID |
|---|---|---|
| Setup | Đơn giản | Cần RBAC role assignment |
| Security | Shared secret, full access | Identity-based, RBAC |
| Rotation | Thủ công | Azure quản lý |
| Audit | Không | ✅ Detailed audit log |
| Conditional access | ❌ | ✅ |
| **Phù hợp** | Development | **Production** |

**DefaultAzureCredential** tự động chọn credential phù hợp:
- Local dev: Azure CLI login hoặc Visual Studio credentials
- Deployed: Managed Identity

Một code path, hoạt động ở mọi environment. Assign RBAC role trước khi dùng:
- `Cosmos DB Built-in Data Reader` — read-only
- `Cosmos DB Built-in Data Contributor` — read/write

---

## Reuse Client — Best Practice Quan Trọng

**Tạo CosmosClient một lần, dùng cho toàn bộ application lifecycle.**

```python
from azure.cosmos import CosmosClient
from azure.identity import DefaultAzureCredential

class CosmosService:
    def __init__(self, endpoint: str):
        credential = DefaultAzureCredential()
        self._client = CosmosClient(endpoint, credential=credential)
        # Cache database và container clients để tránh repeated lookup
        self._database = self._client.get_database_client("aidata")
        self._products_container = self._database.get_container_client("products")

    @property
    def products(self):
        return self._products_container

# Tạo một lần lúc app startup
cosmos_service = CosmosService("https://myaccount.documents.azure.com:443/")
```

**Tại sao quan trọng:** Client duy trì connection pool, cache routing information, handle background operations. Tạo client mới per request → discard cached state → thêm latency, exhaust connections.

---

## Navigate Resource Hierarchy

```python
# get_database_client() và get_container_client() là lightweight handles
# Không verify resource tồn tại — chỉ tạo reference
database = client.get_database_client("productcatalog")
products = database.get_container_client("products")
users = database.get_container_client("users")

# Sử dụng container client
item = products.read_item(item="product-123", partition_key="electronics")
```

Nếu container không tồn tại và bạn thực hiện operation → SDK raise exception.

---

## Tạo Database và Container

### Idempotent (recommended)

```python
# create_if_not_exists — tạo mới hoặc trả về existing
database = client.create_database_if_not_exists(id="productcatalog")

# Container với partition key
products_container = database.create_container_if_not_exists(
    id="products",
    partition_key=PartitionKey(path="/categoryId")
)
```

### Với Manual Throughput

```python
container = database.create_container_if_not_exists(
    id="products",
    partition_key=PartitionKey(path="/categoryId"),
    offer_throughput=400    # 400 RU/s manual
)
```

### Với Autoscale

```python
from azure.cosmos import ThroughputProperties

container = database.create_container_if_not_exists(
    id="inferencecache",
    partition_key=PartitionKey(path="/modelId"),
    offer_throughput=ThroughputProperties(auto_scale_max_throughput=4000)
    # Autoscale 400-4000 RU/s (10%-100% của max)
)
```

---

## CRUD Operations

### Create — insert mới, fail nếu đã tồn tại

```python
from azure.cosmos import exceptions

product = {
    "id": "product-12345",
    "categoryId": "electronics",
    "name": "Smart Speaker",
    "price": 99.99
}

try:
    response = container.create_item(body=product)
except exceptions.CosmosResourceExistsError:
    print("Item already exists — dùng upsert nếu muốn overwrite")
```

### Upsert — insert hoặc replace

```python
# Không quan tâm item có tồn tại hay không
container.upsert_item(body=product)
```

**Dùng upsert cho:** Cache model output, sync data từ external source — khi không cần phân biệt create vs update.

### Replace — update item đã tồn tại

```python
item = container.read_item(item="product-12345", partition_key="electronics")
item["price"] = 149.99

# Replace thông thường
container.replace_item(item=item["id"], body=item)
```

**Replace với Optimistic Concurrency:**

```python
try:
    container.replace_item(
        item=item["id"],
        body=item,
        if_match=item["_etag"]   # Fail nếu item đã bị sửa bởi process khác
    )
except exceptions.CosmosAccessConditionFailedError:
    print("Item was modified concurrently — retry với version mới")
```

### Point Read — hiệu quả nhất

```python
# ~1 RU, route trực tiếp đến đúng partition
item = container.read_item(
    item="product-12345",
    partition_key="electronics"
)

# Nếu không tồn tại
try:
    item = container.read_item(item="product-99999", partition_key="electronics")
except exceptions.CosmosResourceNotFoundError:
    print("Item not found — compute fallback hoặc return default")
```

### Delete

```python
try:
    container.delete_item(item="product-12345", partition_key="electronics")
except exceptions.CosmosResourceNotFoundError:
    print("Item not found")
```

---

## Response Metadata — Monitor RU

```python
container.upsert_item(body=product)

headers = container.client_connection.last_response_headers
ru_charge = headers['x-ms-request-charge']   # RU cost
activity_id = headers['x-ms-activity-id']   # Unique request ID cho support
```

Log `activity_id` khi có lỗi — Azure support dùng để trace request cụ thể.

---

## So sánh CRUD methods

| Method | Behavior | Dùng khi |
|---|---|---|
| `create_item()` | Fail nếu đã tồn tại (409) | Muốn catch duplicate |
| `upsert_item()` | Insert hoặc replace | Cache, sync, không quan tâm existed |
| `replace_item()` | Fail nếu không tồn tại | Update với optimistic concurrency |
| `read_item()` | Point read ~1 RU | Fetch by known id + partition key |
| `delete_item()` | Xoá item | Cleanup, expire cache |

---

## Bản chất bài này là gì?

**Một câu:** SDK pattern cho Cosmos DB = tạo CosmosClient một lần dùng suốt vòng đời app, dùng DefaultAzureCredential thay account key, và biết khi nào dùng `create`, `upsert`, hay `replace`.

### So sánh với các SDK khác và ACA/ACR patterns

| Pattern | Cosmos DB SDK | ACA SDK | ACR / Docker |
|---|---|---|---|
| Auth production | `DefaultAzureCredential` | `--identity system` | `--identity system` |
| Auth dev | Account key / Azure CLI | username/password | az acr login |
| Client reuse | Bắt buộc, tạo 1 lần | N/A (per command) | N/A |
| Idempotent setup | `create_if_not_exists()` | `az containerapp create` | `az acr create` |
| Conflict handling | `CosmosResourceExistsError` | Revision replace | Tag overwrite |

**DefaultAzureCredential là pattern xuyên suốt AI-200:** Cùng một credential pattern dùng cho ACR, ACA, AKS Secret CSI Driver, và bây giờ Cosmos DB. Local dùng `az login`, deployed dùng Managed Identity — không thay đổi code.

**Upsert là default choice cho AI workload:** Cache model output, sync embeddings, import data từ nguồn ngoài — tất cả dùng upsert. Chỉ dùng `create` khi cần detect duplicate (tránh double-process), chỉ dùng `replace` khi cần optimistic concurrency với `_etag`.

---

## Checklist ghi nhớ cho AI-200

- [ ] `CosmosClient` = entry point, manage connection pooling
- [ ] **Account Key** = simple, dev · **DefaultAzureCredential** = Entra ID, production
- [ ] `DefaultAzureCredential`: local dùng CLI credentials, deployed dùng Managed Identity
- [ ] RBAC roles: `Cosmos DB Built-in Data Reader` · `Cosmos DB Built-in Data Contributor`
- [ ] **Tạo CosmosClient một lần** — không tạo per request
- [ ] `create_if_not_exists()` = idempotent, recommended cho initialization
- [ ] `create_item()` fail với `CosmosResourceExistsError` (409)
- [ ] `upsert_item()` = insert hoặc replace, không cần check existed
- [ ] `replace_item()` + `if_match=item["_etag"]` = optimistic concurrency
- [ ] `read_item()` = point read ~1 RU, cần id + partition key
- [ ] `CosmosResourceNotFoundError` khi read/delete item không tồn tại
- [ ] `x-ms-request-charge` = RU cost · `x-ms-activity-id` = unique request ID

---

*Bài tiếp theo: Bài 3 — Query Azure Cosmos DB for NoSQL*
