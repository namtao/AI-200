# Module Assessment — Optimize Vector Search in Azure Database for PostgreSQL

> Khoá: AI-200 · PostgreSQL — Optimize vector search in Azure Database for PostgreSQL

---

## Câu 1

**You're tuning PostgreSQL for a vector search workload with 2 million 1536-dimensional embeddings. Queries are slow and you observe a cache hit ratio of 85%. Which configuration change should you prioritize?**

- Increase `shared_buffers` to keep more data in the PostgreSQL cache ✅
- Decrease `random_page_cost` to encourage more index scans
- Increase `ivfflat.probes` to search more index partitions

**Đáp án: Increase shared_buffers**

Cache hit ratio **85% là rất thấp** cho vector workload. Target là **99%+** — 15% cache miss nghĩa là 15% requests phải đọc từ disk thay vì memory. Với 2M × 1536-dim vectors = ~12 GB vector data, nếu `shared_buffers` không đủ lớn để giữ vectors và indexes trong memory, mỗi query phải đọc disk.

```sql
-- Check current
SHOW shared_buffers;

-- Check cache hit ratio
SELECT sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) AS cache_hit_ratio
FROM pg_statio_user_tables;
```

Starting point: 25% available memory. Tăng dần và monitor.

Tại sao các đáp án còn lại sai:
- **Decrease `random_page_cost`:** Đúng là Azure SSD nên dùng `random_page_cost = 1.1`, nhưng đây không phải priority khi cache hit ratio thấp. Cache miss là bottleneck chính — ngay cả khi planner dùng index nhiều hơn, vẫn phải đọc disk vì data không trong cache.
- **Increase `ivfflat.probes`:** Tăng probes = search nhiều partitions hơn = better recall NHƯNG cũng tốn nhiều compute hơn. Với cache hit ratio 85%, vấn đề là disk I/O, không phải số partitions searched. Tăng probes sẽ làm tình hình tệ hơn (nhiều disk reads hơn).

---

## Câu 2

**You need to create a vector index for a dataset of 5 million product embeddings that receives frequent batch updates (daily full refresh). Build time must be under 30 minutes. Which index configuration should you choose?**

- IVFFlat with `lists` set to `sqrt(rows)` ✅
- HNSW with m=16 and ef_construction=64
- HNSW with m=8 and ef_construction=32

**Đáp án: IVFFlat với lists = sqrt(rows)**

Hai constraint quan trọng: **frequent batch updates** (daily full refresh) và **build time < 30 minutes**.

IVFFlat phù hợp vì:
- **Faster build time** — k-means clustering efficient hơn HNSW graph construction
- **Daily rebuild** không tốn kém như HNSW (HNSW 5M vectors có thể mất 2-6 giờ)
- `lists = sqrt(5,000,000) ≈ 2,236` cho 5M rows

Ước tính build time:
- IVFFlat (lists=2236): 30-60 phút cho 5M vectors → **có thể fit**
- HNSW (m=16, ef=64): 2-6 giờ cho 5M vectors → **quá 30 phút**

Tại sao các đáp án còn lại sai:
- **HNSW m=16, ef=64:** Recall tốt hơn IVFFlat, nhưng build time cho 5M vectors khoảng 2-6 giờ — vượt quá 30 phút requirement. Daily refresh với HNSW là không practical ở scale này.
- **HNSW m=8, ef=32:** Giảm m và ef_construction làm build nhanh hơn nhưng recall kém hơn và vẫn không đảm bảo < 30 phút cho 5M vectors. Reduce parameters không phải là đáp án đúng khi cần fast rebuild.

---

## Câu 3

**Your filtered vector search query filters by `category_id` and then orders by vector similarity. The query plan shows a sequential scan on the products table. What should you check first?**

- Verify that a B-tree index exists on the `category_id` column ✅
- Verify that the vector index uses the same operator class as the query
- Increase `hnsw.ef_search` to expand the search space

**Đáp án: Verify B-tree index on category_id**

Query có cả metadata filter (`category_id`) VÀ vector order. Sequential scan xuất hiện → planner không tìm thấy efficient path.

Metadata filter là bottleneck đầu tiên cần check: nếu không có B-tree index trên `category_id`, PostgreSQL phải sequential scan toàn table để find matching rows trước khi apply vector search.

```sql
-- Check existing indexes
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'products';

-- Add if missing
CREATE INDEX idx_products_category ON products (category_id);

-- Update statistics
ANALYZE products;
```

Tại sao các đáp án còn lại sai:
- **Verify vector index operator class:** Đây là đúng khi **vector index** không được dùng. Nhưng câu hỏi có filter `category_id` → metadata filter cần được evaluated first. Sequential scan thường xảy ra khi không có metadata index — PostgreSQL fall back to seq scan toàn table.
- **Increase `hnsw.ef_search`:** `ef_search` controls recall/speed tradeoff của HNSW, không ảnh hưởng đến việc seq scan hay index scan. Tăng ef_search không fix seq scan problem.

---

## Câu 4

**You're implementing connection management for an AI application that makes 500 vector queries per second during peak traffic. Your Azure Database for PostgreSQL instance supports 1,719 max connections. Which approach should you use?**

- Enable PgBouncer in transaction mode with a pool size appropriate for your application instances ✅
- Create a new database connection for each query request
- Enable PgBouncer in session mode to maintain persistent connections

**Đáp án: PgBouncer transaction mode**

500 queries/second × new connection overhead (50-200ms) = hoàn toàn không khả thi.

**Transaction mode** là best choice cho vector search:
- Client holds connection chỉ trong transaction → connection returned to pool giữa queries
- Tối đa số clients có thể phục vụ với limited connections
- Vector search = simple selects, không cần session state giữa queries

```bash
# Enable PgBouncer
az postgres flexible-server parameter set --name pgbouncer.enabled --value true

# Connect on port 6432
postgresql://user:pass@server.postgres.database.azure.com:6432/db
```

Tại sao các đáp án còn lại sai:
- **New connection per request:** 500 req/s × 50-200ms setup = 25-100 seconds chỉ để establish connections mỗi giây. Impossible. Sẽ exhaust max_connections (1,719) và cause connection failures.
- **Session mode:** Client giữ connection cả session (until disconnect). Minimal connection saving — nếu có 500 concurrent sessions, vẫn cần 500 connections. Session mode không giải quyết high-concurrency problem.

---

## Câu 5

**You're scaling a recommendation engine that currently runs on a General Purpose 8 vCore instance. CPU utilization averages 75% and P95 query latency is 150 ms, but you need to achieve sub-50ms latency. What scaling approach should you try first?**

- Upgrade to a Memory Optimized tier with more vCores ✅
- Add read replicas to distribute query load
- Implement application-level caching with Azure Cache for Redis

**Đáp án: Upgrade to Memory Optimized**

CPU 75% + P95 latency 150ms (cần <50ms) = **single-server performance bottleneck**, không phải throughput issue.

Memory Optimized tier giải quyết:
- **8 GB/vCore** (vs 4 GB/vCore ở GP) → giữ HNSW indexes và hot data trong memory
- More vCores → parallel distance calculations
- Reduced disk I/O → lower latency per query

Đây là **vertical scaling** — cải thiện single-query latency. Thử trước khi horizontal scaling.

Tại sao các đáp án còn lại sai:
- **Read replicas:** Distribute total query load, không reduce individual query latency. Nếu single query mất 150ms, adding replica vẫn mất 150ms per query. Replicas help khi server bị overwhelmed bởi volume, không phải khi single query is slow.
- **Redis caching:** Caching giúp repeated identical queries, không phải unique vector similarity searches. Recommendation engine thường có unique query embeddings per user — không cacheable hiệu quả. Caching không address computational bottleneck.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Cache hit ratio 85% — slow queries | Tăng `shared_buffers` |
| 2 | 5M vectors, daily refresh, build < 30 min | IVFFlat với `lists = sqrt(rows)` |
| 3 | Sequential scan với category filter | Verify B-tree index trên `category_id` |
| 4 | 500 queries/sec, high concurrency | PgBouncer transaction mode |
| 5 | P95 latency 150ms, cần < 50ms | Upgrade Memory Optimized tier |

---

*Module hoàn thành. PostgreSQL Learning Path kết thúc.*

---

[← Bài 4](./postgresql-m3-bai4-scale-connections.md) · [🏠 Mục lục](../README.md) · [Tổng hợp →](./postgresql-summary.md)
