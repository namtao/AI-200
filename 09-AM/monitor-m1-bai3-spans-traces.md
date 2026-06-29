# Bài 3 — Configure Spans and Traces

> Khoá: AI-200 · Azure Monitor — Instrument an app with OpenTelemetry

---

## Create Custom Spans

```python
from azure.monitor.opentelemetry import configure_azure_monitor
from opentelemetry import trace

configure_azure_monitor()

tracer = trace.get_tracer("embedding-service")    # Identify instrumentation source

with tracer.start_as_current_span("GenerateEmbedding") as span:
    span.set_attribute("embedding.model", "text-embedding-ada-002")
    span.set_attribute("embedding.token_count", token_count)
    embedding = generate_embedding(input_text)
    # Span ends automatically khi block exits (kể cả khi exception)
```

`start_as_current_span` = context manager. Span auto-ends khi block exits.

---

## Span Attributes

```python
with tracer.start_as_current_span("SearchVectorIndex") as span:
    span.set_attribute("search.index_name", "product-docs")
    span.set_attribute("search.query_vector_dimension", 1536)
    span.set_attribute("search.top_k", 10)
    results = search_index(embedding)
    span.set_attribute("search.result_count", len(results))
```

- **Span attributes** = operation-specific, set within span scope
- **Resource attributes** = service-level, set at startup (`service.name`, etc.)
- Attributes → `customDimensions` column trong App Insights → queryable bằng KQL

**Naming convention:** Namespaced keys (`embedding.model`, `search.result_count`, `llm.token_count`). Không dùng generic (`value`, `data`).

---

## Span Kinds và App Insights Mapping

| SpanKind | App Insights table |
|---|---|
| `SERVER` | **requests** |
| `CLIENT` | **dependencies** |
| `INTERNAL` (default) | dependencies |
| `PRODUCER` | dependencies |
| `CONSUMER` | **requests** |

```python
from opentelemetry.trace import SpanKind

# Outgoing LLM API call = CLIENT
with tracer.start_as_current_span("CallLlmApi", kind=SpanKind.CLIENT) as span:
    span.set_attribute("llm.provider", "azure-openai")
    span.set_attribute("llm.model", "gpt-4o")
    response = call_llm(prompt)
    span.set_attribute("llm.response_tokens", response.usage.completion_tokens)
```

---

## Error Handling

SDK tự động record exception khi propagate ra khỏi span block. Để manual control:

```python
from opentelemetry.trace import Status, StatusCode

with tracer.start_as_current_span("GenerateEmbedding") as span:
    try:
        embedding = generate_embedding(input_text)
        span.set_status(Status(StatusCode.OK))
    except Exception as ex:
        span.record_exception(ex)
        span.set_status(Status(StatusCode.ERROR, "Embedding generation failed"))
        raise
```

---

## Nested Spans (Parent-child Hierarchy)

```python
tracer = trace.get_tracer("llm-orchestrator")

def process_query(query: str) -> str:
    with tracer.start_as_current_span("ProcessQuery", kind=SpanKind.SERVER) as root:
        root.set_attribute("query.length", len(query))

        with tracer.start_as_current_span("GenerateEmbedding") as embed:
            embed.set_attribute("embedding.model", "text-embedding-ada-002")
            embedding = embedding_service.generate(query)

        with tracer.start_as_current_span("SearchVectorIndex") as search:
            search.set_attribute("search.top_k", 5)
            results = vector_search.search(embedding, top_k=5)
            search.set_attribute("search.result_count", len(results))

        with tracer.start_as_current_span("CallLlm", kind=SpanKind.CLIENT) as llm:
            llm.set_attribute("llm.prompt_tokens", count_tokens(prompt))
            response = llm_client.get_completion(build_prompt(query, results))
            llm.set_attribute("llm.response_tokens", response.usage.completion_tokens)
            return response.content
```

SDK tự động set parent — không cần pass references. Context tracked via Python context variables.

Kết quả: Waterfall chart trong App Insights. ProcessQuery → GenerateEmbedding, SearchVectorIndex, CallLlm. Bottleneck visible ngay.

---

## Bản chất bài này là gì?

**Một câu:** Span là đơn vị đo lường của distributed tracing — mỗi operation quan trọng trong AI pipeline (embedding, vector search, LLM call) cần một span riêng với attributes mô tả đủ để debug sau.

### So sánh với Prometheus metrics và ELK Stack logs

| | Prometheus Metrics | ELK Stack Logs | OTel Spans/Traces |
|---|---|---|---|
| **Granularity** | Aggregate (count, histogram) | Per-event text records | Per-operation timed records với hierarchy |
| **Causality** | Không (chỉ biết "tăng") | Partial (correlation IDs thủ công) | **Built-in** (parent-child, trace context) |
| **AI pipeline debug** | "Có bao nhiêu embedding calls" | "Log message từ embedding service" | "Embedding call X trong request Y mất 2s" |
| **Custom metadata** | Labels (limited cardinality) | Log fields | **span attributes** → `customDimensions` |
| **Parent-child** | Không | Không | **Automatic** via context propagation |

**Default SpanKind = INTERNAL → dependencies table.** Nếu quên set `kind=SpanKind.CLIENT` cho LLM call → vẫn vào dependencies nhưng semantics sai (CLIENT = outgoing call rõ ràng hơn).

**Exam trap:** Nested spans — SDK tự động set parent từ Python context variables, KHÔNG cần pass tracer hoặc span references giữa functions. Đây là điểm hay bị hiểu sai.

---

## Checklist ghi nhớ cho AI-200

- [ ] `tracer = trace.get_tracer("name")` — once per service/component
- [ ] `start_as_current_span` context manager — auto-ends khi block exits
- [ ] `span.set_attribute(key, value)` → `customDimensions` trong App Insights
- [ ] Resource attributes = service-level · Span attributes = operation-specific
- [ ] CLIENT → dependencies table · SERVER/CONSUMER → requests table
- [ ] Default SpanKind = **INTERNAL** → dependencies
- [ ] Nested spans: SDK auto-sets parent từ context — không pass references
- [ ] `span.record_exception(ex)` + `span.set_status(Status(StatusCode.ERROR, msg))`

---

*Bài tiếp theo: Bài 4 — Export telemetry to Azure Monitor*
