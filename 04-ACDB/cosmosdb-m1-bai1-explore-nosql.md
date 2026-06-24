# Bài 1 — Explore Azure Cosmos DB for NoSQL

> Khoá: AI-200 · Cosmos DB — Build queries for Azure Cosmos DB for NoSQL

---

## Cosmos DB là gì và khi nào dùng?

Azure Cosmos DB for NoSQL là **globally distributed, schemaless document database** — lưu trữ JSON document với automatic indexing và low-latency performance.

**Khác với relational database:**
- Schema-agnostic: document trong cùng container có thể có cấu trúc khác nhau
- Không cần migration khi thay đổi data model
- Scale throughput độc lập với storage
- Built-in global distribution

**Với AI application:** Phù hợp cho user profiles, product catalogs, inference results cache, interaction logs — những data có structure thay đổi theo thời gian và cần truy cập millisecond.

---

## Resource Hierarchy — 4 tầng

```
Account (top-level, DNS endpoint)
└── Database (logical namespace)
    └── Container (unit of scalability, partition key)
        └── Item (JSON document)
```

### Account

- Top-level management unit
- DNS endpoint: `https://<account-name>.documents.azure.com:443/`
- Config: consistency level, geo-replication, network access
- Một subscription có nhiều account (prod/staging/dev tách nhau)

### Database

- Logical namespace nhóm container liên quan
- Tối đa 500 databases + containers combined per account
- Scope cho **shared throughput** — nhiều container chia sẻ RU/s

### Container

- **Fundamental unit of scalability**
- Chứa JSON documents
- **Schema-agnostic** — item trong container có thể có structure khác nhau
- Yêu cầu khai báo **partition key** khi tạo
- Không thể đổi partition key sau khi tạo

### Item

- JSON document trong container
- Yêu cầu `id` property (không auto-generate)
- Combination `id` + partition key = unique identifier across container

---

## Partition Key — Quyết định quan trọng nhất

Partition key xác định cách data được phân phối across physical partitions. **Không thể thay đổi sau khi tạo container** — phải migrate data.

### Tiêu chí chọn partition key tốt

- **Tồn tại trong mọi document**
- **High cardinality** — nhiều distinct values
- **Match query pattern** — query thường filter theo field nào thì đó là partition key tốt

**Ví dụ tốt cho AI application:**
- `userId` — user-specific data, mỗi user là một partition, spread tốt
- `categoryId` — product catalog, query thường filter theo category
- `modelId` — inference cache, phân nhóm theo model

**Tránh:**
- Boolean (`isActive`) → chỉ 2 partition → hot partition
- Date/timestamp → hot partition trong high-activity period
- Giá trị ít distinct (status với 3-4 giá trị)

### Tạo container với partition key (Python)

```python
from azure.cosmos import CosmosClient, PartitionKey

database = client.get_database_client("productcatalog")

container = database.create_container_if_not_exists(
    id="products",
    partition_key=PartitionKey(path="/categoryId"),  # path đến JSON property
    offer_throughput=400
)
```

Có thể dùng nested property: `PartitionKey(path="/metadata/region")`

---

## Throughput — Request Units (RU/s)

**RU (Request Unit)** là đơn vị đo throughput, abstract hóa CPU + memory + IOPS thành một metric.

### RU cost của các operation (1-KB item)

| Operation | RU cost |
|---|---|
| Point read (by id + partition key) | ~1 RU |
| Write (1-KB item) | ~5-10 RU |
| Simple filtered query | ~3-5 RU |
| Aggregate query (scan nhiều item) | Có thể hàng trăm RU |

### Các yếu tố ảnh hưởng RU

- **Item size** — document lớn hơn = tốn RU hơn
- **Indexing policy** — nhiều index = write tốn RU hơn, query rẻ hơn
- **Consistency level** — stronger consistency = tốn RU hơn
- **Cross-partition query** — fan out to all partitions = đắt hơn single-partition

### Provisioning levels

**Database-level throughput** — chia sẻ across containers, tốt khi usage pattern variable.

**Container-level throughput** — dedicated cho một container, predictable performance.

### Throughput modes

**Manual:** Fix số RU/s bạn specify. Min: 400 RU/s.

**Autoscale:** Scale giữa 10% và 100% của configured maximum. Minimum autoscale max: 1000 RU/s. Tốt cho AI workload với traffic spike bất ngờ.

---

## System Properties của Item

Mỗi item có system-generated properties:

| Property | Ý nghĩa |
|---|---|
| `_rid` | Internal resource identifier |
| `_self` | Unique URI để address resource trực tiếp |
| `_etag` | Entity tag cho **optimistic concurrency** — thay đổi mỗi khi item update |
| `_ts` | Unix timestamp của lần modify cuối (seconds) |
| `_attachments` | Legacy feature |

### Optimistic Concurrency với `_etag`

Khi update item, include `_etag` như condition. Nếu item đã bị thay đổi bởi process khác → update fail với conflict error → không mất data.

Quan trọng cho AI application cache model output và cần consistent update.

---

## Monitor RU Consumption

Mỗi operation trả về RU charge trong response header:

```python
container.upsert_item(body=product)
headers = container.client_connection.last_response_headers
ru_charge = headers['x-ms-request-charge']
```

**Azure Monitor + Cosmos DB Insights** cung cấp dashboard track RU consumption over time — identify expensive operation để optimize.

---

## Bản chất bài này là gì?

**Một câu:** Cosmos DB for NoSQL là schemaless distributed document database — bạn lưu JSON với bất kỳ cấu trúc nào, scale throughput độc lập với storage, và partition key là quyết định thiết kế không thể thay đổi quan trọng nhất.

### So sánh SQL vs Cosmos DB NoSQL

| Khái niệm | SQL Server / PostgreSQL | Cosmos DB NoSQL |
|---|---|---|
| Đơn vị lưu trữ | Row trong Table | Item (JSON document) |
| Schema | Rigid, migration required | Schema-agnostic, documents khác nhau trong cùng container |
| Hierarchy | Server → Database → Table | Account → Database → Container → Item |
| Scale throughput | Vertical (upgrade server) | Horizontal, RU/s config per container |
| Distribution | Manual (Always On, etc.) | Built-in global replication |
| Query language | SQL (full) | SQL dialect (subset, không có JOINs across containers) |
| Concurrency | Row-level locking | Optimistic concurrency via `_etag` |
| Primary key | Composite PK | `id` + partition key |

**RU là abstraction không có trong SQL thế giới:** Thay vì quan tâm CPU/IOPS/memory riêng lẻ, RU gộp tất cả thành một metric. Point read ~1 RU, write ~5-10 RU — dễ estimate cost hơn.

**Partition key là quyết định không thể thay đổi:** Phải chọn trước khi có data. Chọn sai → hot partition (một partition nhận quá nhiều traffic) → throttling → phải migrate toàn bộ data vào container mới.

---

## Checklist ghi nhớ cho AI-200

- [ ] Hierarchy: **Account → Database → Container → Item**
- [ ] Account DNS: `https://<name>.documents.azure.com:443/`
- [ ] **Partition key không thể thay đổi** sau khi tạo container
- [ ] Partition key tốt: high cardinality, tồn tại trong mọi doc, match query pattern
- [ ] Tránh: boolean, timestamp (hot partition), ít distinct values
- [ ] Point read ~1 RU · Write ~5-10 RU · Cross-partition query = đắt
- [ ] Manual throughput min: **400 RU/s** · Autoscale min max: **1000 RU/s**
- [ ] Autoscale: 10%–100% của configured max
- [ ] `_etag` dùng cho **optimistic concurrency** — prevent lost updates
- [ ] `_ts` = Unix timestamp của last modification
- [ ] `x-ms-request-charge` header = RU cost của operation

---

*Bài tiếp theo: Bài 2 — Implement the Azure Cosmos DB for NoSQL SDK*
