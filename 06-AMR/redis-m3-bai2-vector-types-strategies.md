# Bài 2 — Choose Vector Types and Indexing Strategies

> Khoá: AI-200 · Redis — Implement vector storage in Azure Managed Redis

---

## Vector Data Types

| Type | Bytes/dim | 1536-dim vector | Dùng khi |
|---|---|---|---|
| **FLOAT32** | 4 bytes | **6 KB** | **Standard — hầu hết AI applications** |
| FLOAT64 | 8 bytes | 12 KB | Specialized research, maximum precision |

FLOAT64 doubles memory và slows queries — extra precision không có meaningful improvement cho AI embeddings vì models có inherent noise. **Dùng FLOAT32.**

---

## Dimension Matching

```python
embedding_models = {
    "small":  384,
    "medium": 768,
    "large":  1536,   # OpenAI text-embedding-ada-002
    "xlarge": 3072,
}
```

DIM phải match **chính xác** model output — mismatch = ingestion fail hoặc query error.

---

## Distance Metrics

| Metric | Đo gì | Dùng khi |
|---|---|---|
| **COSINE** | Angle giữa vectors (ignoring magnitude) | **Text embeddings — OpenAI, Cohere, Sentence Transformers** |
| L2 | Straight-line distance (direction + magnitude) | Image embeddings, spatial data |
| IP | Dot product | Pre-normalized embeddings only |

**Key point:** Phải match metric với cách embedding model được trained. COSINE với text embedding model trained cho L2 = mathematically đúng nhưng semantically vô nghĩa.

```python
# Text embeddings
VectorField("embedding", "HNSW", {"TYPE": "FLOAT32", "DIM": 1536, "DISTANCE_METRIC": "COSINE"})

# Image embeddings
VectorField("image_embedding", "HNSW", {"TYPE": "FLOAT32", "DIM": 768, "DISTANCE_METRIC": "L2"})
```

---

## Indexing Algorithms

### FLAT — Exact search

Brute-force: so sánh query với **tất cả** vectors. 100% accuracy nhưng O(n) — không scale.

```python
VectorField("embedding", "FLAT", {"TYPE": "FLOAT32", "DIM": 1536, "DISTANCE_METRIC": "COSINE"})
```

Dùng khi: <10,000 vectors, development/prototyping, cần 100% exact results.

### HNSW — Approximate search

Multi-layer graph: như highway system (top layer = interstate highways, bottom = local streets). Query navigate qua graph, chỉ examine small fraction of total vectors.

```python
VectorField("embedding", "HNSW", {"TYPE": "FLOAT32", "DIM": 1536, "DISTANCE_METRIC": "COSINE"})
```

Dùng khi: >10,000 vectors, production, cần <10ms latency, 95-99% accuracy OK.

### EF_RUNTIME — Query-time tuning

```python
for ef_runtime in [50, 100, 200]:
    query = Query(f"*=>[KNN 10 @embedding $query_vec EF_RUNTIME {ef_runtime} AS score]").dialect(2)
    # Measure latency và accuracy tradeoff
```

Default ≈ 10 (speed). Increase đến 100-200 cho better recall.

---

## Quick Configuration Guide

| Use Case | Data Type | Metric | Algorithm |
|---|---|---|---|
| Text search (small) | FLOAT32 | COSINE | FLAT |
| Text search (large) | FLOAT32 | COSINE | HNSW |
| Image similarity | FLOAT32 | L2 | HNSW |
| Recommendations | FLOAT32 | COSINE | HNSW |

---

## Bản chất bài này là gì?

**Một câu:** FLOAT32 + COSINE + HNSW là combo mặc định cho text embedding workloads — FLOAT64 doubles memory vô ích, wrong distance metric = semantically wrong results dù mathematically valid.

### So sánh FLOAT32 vs FLOAT64 cho AI embeddings

| | FLOAT32 | FLOAT64 |
|---|---|---|
| Bytes per dimension | 4 bytes | 8 bytes |
| 1536-dim vector size | 6 KB | 12 KB |
| 1M vectors memory | 6 GB | 12 GB |
| AI model compatibility | ✅ All major models | ✅ Nhưng không cần |
| Meaningful precision gain | ❌ | Models có inherent noise |
| Phù hợp | Production AI | Research scenarios đặc biệt |

**Wrong metric = mathematically valid but semantically wrong:** COSINE với text embedding trained cho L2 trả về valid numbers nhưng wrong semantic similarity. Model không recognize "cat" và "feline" là similar vì metric không match training objective.

**Exam trap — EF_RUNTIME không phải max results:** `KNN 10` = trả về 10 results. `EF_RUNTIME 200` = explore 200 graph nodes để tìm candidates. Hai parameter khác nhau hoàn toàn.

---

## Checklist ghi nhớ cho AI-200

- [ ] **FLOAT32** standard — 4 bytes/dim, 6 KB cho 1536-dim
- [ ] FLOAT64 = doubles memory, no meaningful accuracy improvement cho AI
- [ ] Common dimensions: 384, 768, 1024, **1536** (ada-002), 3072
- [ ] **COSINE** cho text embeddings (OpenAI, Cohere) · **L2** cho image
- [ ] Metric phải match embedding model training objective
- [ ] FLAT = 100% exact, slow · HNSW = 95-99% approximate, sub-10ms
- [ ] FLAT: <10K vectors · HNSW: >10K vectors production
- [ ] EF_RUNTIME controls HNSW graph exploration depth — higher = better recall + slower

---

[← Bài 1](./redis-m3-bai1-index-query-vectors.md) · [🏠 Mục lục](../README.md) · [Bài 3 →](./redis-m3-bai3-data-structures.md)
