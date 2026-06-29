# Bài 1 — Explore Azure Managed Redis

> Khoá: AI-200 · Redis — Implement data operations in Azure Managed Redis

---

## Azure Managed Redis là gì?

In-memory data store dựa trên **Redis Enterprise** software. Microsoft quản lý hạ tầng, hosted trên Azure. Redis Enterprise cải thiện performance và reliability so với community Redis, đồng thời duy trì compatibility.

**Lợi ích cho AI app:** Sub-millisecond latency cho user profiles, session data, conversation history, cached model results — không phải query database mỗi lần.

---

## 3 Common Caching Strategies

### 1. Data Cache — Cache-aside Pattern

Database thường quá lớn để load toàn bộ vào cache. Cache-aside (lazy loading):

```
App cần data
    → Check Redis
    → Cache HIT: trả về từ Redis (fast)
    → Cache MISS: query database → lưu vào Redis → trả về
```

Tính năng hỗ trợ:
- **TTL (Time-to-live)** — tự động remove stale data
- **Eviction policies** — manage memory khi cache đầy
- **Active-Active geo-replication** — sync data across regions

### 2. Content Cache

Static content (headers, footers, navigation, banners) ít thay đổi → ideal cho caching.

- Sub-millisecond access thay vì regenerate từ backend
- Giảm server load → handle nhiều concurrent users hơn
- Giảm số web servers cần thiết → lower infrastructure costs

### 3. Session Store

Lưu session data (shopping cart, user preferences, auth tokens, conversation context):

- Chỉ lưu session ID trong cookie → Redis giữ full session data
- Sub-millisecond latency, scale đến millions of concurrent sessions
- Automatic session expiration, replication, framework integration

---

## 4 Tiers

**3 tiers in-memory:**

| Tier | Memory:vCPU | Phù hợp |
|---|---|---|
| **Memory Optimized** | 8:1 | Memory-intensive, dev/test, lower throughput OK |
| **Balanced** | 4:1 | Standard production workloads |
| **Compute Optimized** | 2:1 | Maximum throughput, performance-intensive |

**1 tier in-memory + disk:**

| Tier | Description |
|---|---|
| **Flash Optimized** (preview) | Tự động chuyển ít-accessed data từ RAM sang NVMe — giảm cost cho large datasets |

---

## Bản chất bài này là gì?

**Một câu:** Azure Managed Redis là in-memory store dựa trên Redis Enterprise, Microsoft-managed, hỗ trợ 3 caching patterns và 4 tiers với tradeoff giữa memory, compute, và cost.

### So sánh với Azure Cache for Redis

| | Azure Cache for Redis | Azure Managed Redis |
|---|---|---|
| Engine | Community Redis | Redis Enterprise |
| Default port | 6380 | **10000** |
| Geo-replication | Basic | Active-Active |
| Flash tier | ❌ | ✅ (preview) |
| Phù hợp | General caching | AI/high-performance |

**Cache-aside là pattern phổ biến nhất:** App check Redis trước → HIT trả về ngay, MISS mới query DB rồi store vào Redis.

**Exam trap — 4 tiers, không phải 3:** Memory Optimized (8:1), Balanced (4:1), Compute Optimized (2:1), và Flash Optimized (disk). Flash Optimized là tier thứ 4 duy nhất dùng NVMe disk.

---

## Checklist ghi nhớ cho AI-200

- [ ] Azure Managed Redis = Redis Enterprise, managed by Microsoft
- [ ] 3 caching strategies: **Data cache** (cache-aside) · **Content cache** · **Session store**
- [ ] Cache-aside: check Redis → HIT return · MISS query DB → store → return
- [ ] TTL = auto-expire stale data · Eviction = manage memory khi đầy
- [ ] Active-Active geo-replication = sync across regions
- [ ] 4 tiers: Memory Optimized (8:1) · Balanced (4:1) · Compute Optimized (2:1) · Flash Optimized (disk)
- [ ] Memory Optimized = memory-intensive, dev/test
- [ ] Compute Optimized = max throughput
- [ ] Flash Optimized = large dataset, cost-effective

---

*Bài tiếp theo: Bài 2 — Client libraries and development best practices*
