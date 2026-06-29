# Bài 4 — Implement Retrieval Patterns for RAG Pipelines

> Khoá: AI-200 · PostgreSQL — Implement vector search with Azure Database for PostgreSQL

---

## RAG Architecture với PostgreSQL

```
User query → Embedding model → Query vector
                                    ↓
PostgreSQL (pgvector) ← Similarity search → Top-k chunks
                                    ↓
LLM (GPT-4...) + Chunks as context → Generated answer with citations
```

PostgreSQL với pgvector = retriever. Retrieval quality → LLM answer quality.

---

## Schema cho RAG — 2 Tables

```sql
-- Source documents (metadata)
CREATE TABLE source_documents (
    id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    source_url TEXT,
    document_type TEXT,
    ingested_at TIMESTAMPTZ DEFAULT NOW()
);

-- Chunks với embeddings
CREATE TABLE document_chunks (
    id SERIAL PRIMARY KEY,
    document_id INTEGER REFERENCES source_documents(id) ON DELETE CASCADE,
    chunk_index INTEGER NOT NULL,
    content TEXT NOT NULL,
    embedding vector(1536),
    token_count INTEGER,
    section_title TEXT,
    page_number INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (document_id, chunk_index)
);
```

**Lợi ích 2-table design:**
- Document metadata lưu một lần, không duplicate per chunk
- Delete document = automatically delete chunks (CASCADE)
- Independent retrieval: chunk alone hoặc reconstruct document

### Indexes cho RAG patterns

```sql
-- Vector index
CREATE INDEX doc_chunks_embedding_idx ON document_chunks
USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);

-- B-tree cho JOIN và filter
CREATE INDEX doc_chunks_document_id_idx ON document_chunks (document_id);

-- Composite cho context window queries (chunk ordering)
CREATE INDEX doc_chunks_doc_chunk_idx ON document_chunks (document_id, chunk_index);
```

---

## Chunking Strategies

| Strategy | Mô tả | Dùng khi |
|---|---|---|
| **Fixed-size** | Split mỗi N characters/tokens | Content không có clear structure, consistent token counts |
| **Semantic** | Split tại paragraph/section boundaries | Structured documents (legal clauses, FAQ) |
| **Overlapping** | Include N chars overlap giữa chunks | Concepts span chunk boundaries |

---

## Context Window Retrieval

```sql
WITH matched_chunks AS (
    -- Top-k most similar chunks
    SELECT id, document_id, chunk_index, embedding <=> $1 AS distance
    FROM document_chunks
    ORDER BY embedding <=> $1
    LIMIT 3
),
context_window AS (
    -- Expand: include adjacent chunks (±1)
    SELECT DISTINCT dc.id, dc.document_id, dc.chunk_index, dc.content,
           dc.token_count, mc.distance
    FROM matched_chunks mc
    JOIN document_chunks dc ON dc.document_id = mc.document_id
        AND dc.chunk_index BETWEEN mc.chunk_index - 1 AND mc.chunk_index + 1
),
token_limited AS (
    -- Track cumulative tokens, respect LLM context limit
    SELECT cw.*, sd.title AS source_title, sd.source_url,
           SUM(cw.token_count) OVER (ORDER BY cw.distance, cw.chunk_index) AS cumulative_tokens
    FROM context_window cw
    JOIN source_documents sd ON cw.document_id = sd.id
)
SELECT id, content, source_title, source_url, distance
FROM token_limited
WHERE cumulative_tokens <= 3000    -- Token budget
ORDER BY distance, chunk_index;
```

**Điều chỉnh theo use case:**
- Legal specific clauses: 5 matches, no context window
- Customer support open questions: 3 matches + 2 adjacent chunks

---

## Citation-aware Retrieval

Group chunks theo source document để tránh duplicate citations:

```sql
WITH ranked_chunks AS (
    SELECT
        dc.*,
        sd.title AS document_title,
        sd.source_url,
        dc.embedding <=> $1 AS distance,
        ROW_NUMBER() OVER (PARTITION BY dc.document_id ORDER BY dc.embedding <=> $1) AS rank_in_doc
    FROM document_chunks dc
    JOIN source_documents sd ON dc.document_id = sd.id
    WHERE dc.embedding <=> $1 < 0.5    -- Quality threshold
)
SELECT
    document_id,
    document_title,
    source_url,
    array_agg(content ORDER BY chunk_index) AS chunks,    -- All relevant chunks from doc
    MIN(distance) AS best_distance
FROM ranked_chunks
WHERE rank_in_doc <= 3    -- Max 3 chunks per document
GROUP BY document_id, document_title, source_url
ORDER BY best_distance
LIMIT 5;
```

Format citation: `"The employer may terminate..." — Employment Agreement, Section 4.2, Page 3`

---

## Document Update Strategy

**Replace (simpler):** Delete source document → chunks auto-deleted (CASCADE) → reingest.

```sql
DELETE FROM source_documents WHERE id = :doc_id;
-- Rồi insert lại
```

HNSW handles deletions gracefully. IVFFlat có thể cần rebuild sau significant changes.

---

## RAG Retrieval Quality Metrics

| Metric | Ý nghĩa | Improve bằng |
|---|---|---|
| **Precision** | % retrieved chunks thực sự relevant | Tighter threshold, fewer chunks |
| **Recall** | % relevant chunks được retrieved | Looser threshold, more chunks, tune `ef_search` |
| **MRR** | First relevant result rank cao không | Better embedding model, query preprocessing |

**Evaluation dataset:** 20-50 representative queries + human relevance labels → measure metrics khi thay đổi parameters.

**Parameters ảnh hưởng precision/recall:**
- Chunk size (smaller = precise, larger = better recall)
- Chunk overlap (more = better recall at boundaries)
- Distance threshold (tighter = precise, looser = recall)
- `ef_search` (HNSW) / `probes` (IVFFlat) — higher = better recall + slower

---

## Bản chất bài này là gì?

**Một câu:** RAG với pgvector là 2-table design (source_documents + document_chunks với CASCADE) kết hợp vector search, context window expansion, và citation-aware grouping trong một SQL query — thay vì orchestrate nhiều service calls như với Pinecone + separate metadata store.

### So sánh với LangChain PGVector vs Pinecone + metadata vs Azure AI Search + Blob

| | pgvector RAG (custom) | LangChain PGVectorStore | Pinecone + PostgreSQL | Azure AI Search |
|---|---|---|---|---|
| Chunk + metadata join | ✅ SQL JOIN | ✅ Managed | 2 separate calls | ✅ Single query |
| Context window (adjacent chunks) | ✅ SQL BETWEEN | ❌ (app logic) | ❌ (app logic) | ❌ (app logic) |
| Citation grouping | ✅ `GROUP BY` + `array_agg` | ❌ | ❌ | Partial |
| Token budget tracking | ✅ `SUM() OVER (ORDER BY)` | ❌ | ❌ | ❌ |
| Atomic update (doc + chunks) | ✅ CASCADE DELETE | ✅ | ❌ (2 systems) | ✅ |
| Hybrid search | Manual SQL | ❌ | ❌ Native | ✅ Built-in RRF |
| Schema flexibility | Custom | Fixed schema | Custom | Custom |
| Operational complexity | Medium | Low | High (2 systems) | Low |

**2-table design với CASCADE là thiết kế không obvious từ vector DB mindset:** Dedicated vector DBs (Pinecone, Qdrant) không có relational model — xóa document phải manual xóa từng chunk bằng filter query. PostgreSQL `ON DELETE CASCADE` = xóa `source_documents` row → tất cả `document_chunks` tự xóa atomically. Nếu xóa giữa chừng bị interrupted, không có orphan chunks.

**Context window expansion (±1 adjacent chunks) không thể trong Pinecone:** Pinecone không biết "chunk_index" relationship giữa chunks của cùng document — chỉ biết similarity score. SQL `BETWEEN mc.chunk_index - 1 AND mc.chunk_index + 1` kết hợp với `JOIN ON document_id` là pattern native của relational DB, không phải vector DB. Đây là lý do tại sao RAG với complex context management thuộc về PostgreSQL hơn dedicated vector store.

---

## Checklist ghi nhớ cho AI-200

- [ ] RAG: query embedding → retrieve chunks → LLM generates answer
- [ ] 2-table design: `source_documents` + `document_chunks` với CASCADE
- [ ] B-tree index trên `document_id` VÀ composite `(document_id, chunk_index)`
- [ ] 3 chunking strategies: fixed-size, semantic, overlapping
- [ ] Context window: adjacent chunks (±1) cho narrative/connected content
- [ ] Token budget: `SUM(token_count) OVER (ORDER BY ...)` — cumulative tracking
- [ ] Citation: group by document, max N chunks per doc, `array_agg(content)`
- [ ] Delete source document → CASCADE deletes chunks automatically
- [ ] RAG metrics: Precision, Recall, MRR
- [ ] Evaluation: 20-50 labeled queries → measure before/after changes

---

*Module Assessment tiếp theo*
