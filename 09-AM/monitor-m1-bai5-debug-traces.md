# Bài 5 — Debug Distributed Flows with Trace Data

> Khoá: AI-200 · Azure Monitor — Instrument an app with OpenTelemetry

---

## Application Map

Visual topology: nodes = cloud roles, edges = calls giữa services.

Hiển thị per node/edge: average response time, request count, failure rate. Identify bottleneck nhanh:
- Vector search node: avg 5s · Các nodes khác: avg 500ms → vector search là vấn đề

Navigate: node → drill into requests · edge → call characteristics giữa 2 services.

---

## End-to-end Transaction View (Waterfall)

Waterfall chart: root span ở trên, child spans nested bên dưới, horizontal timeline.

**Đọc waterfall:**
- Total 10s, LLM span 8s → LLM là bottleneck
- Embedding span consistent 200ms, vector search 100ms-5s → intermittent issue ở vector search

Select span → details: name, duration, status code, custom attributes, exception details.

---

## KQL Queries

Tất cả tables share `operation_Id` = OpenTelemetry trace ID → join spans từ cùng trace.

### Slow requests by service

```kql
requests
| where duration > 3000
| summarize count(), avg(duration) by cloud_RoleName
| order by avg_duration desc
```

### Join requests với downstream dependencies

```kql
requests
| where duration > 5000
| join kind=inner (dependencies) on operation_Id
| project timestamp, requestName = name, requestDuration = duration,
    dependencyName = name1, dependencyDuration = duration1,
    dependencyTarget = target
| order by requestDuration desc
```

`operation_Id` join = link tất cả spans trong same distributed trace.

### Phân tích custom attributes (AI-specific)

```kql
dependencies
| where name == "GenerateEmbedding"
| extend tokenCount = toint(customDimensions["embedding.token_count"]),
    model = tostring(customDimensions["embedding.model"])
| summarize avg(duration), percentile(duration, 95), avg(tokenCount) by model
| order by avg_duration desc
```

`customDimensions["attribute.name"]` → access span attributes.

---

## AI-specific Diagnostic Patterns

| Pattern | Symptoms | Diagnose |
|---|---|---|
| **Embedding generation timeout** | Slow/fail embedding spans | Query duration > threshold, error status codes, filter by `embedding.model` |
| **Vector search cold start** | High duration variability on first queries | Clusters of slow spans after inactivity periods |
| **LLM token rate limiting** | HTTP 429, unusually long durations | Check `llm.prompt_tokens`, `llm.response_tokens` attributes |
| **Context window overflow** | LLM API error | `llm.prompt_tokens` approaching max → API error |

---

## Proactive Monitoring

- **Alerts** trong App Insights cho latency thresholds on specific operations
- **Workbooks** cho AI pipeline health dashboards
- **Correlate traces với custom metrics** để detect trends

---

## Bản chất bài này là gì?

**Một câu:** Application Map cho thấy topology, waterfall cho thấy timing, KQL cho thấy patterns — kết hợp 3 công cụ này để debug distributed AI pipeline từ symptom đến root cause.

### So sánh với Jaeger UI và Grafana Tempo

| | Jaeger UI | Grafana Tempo | Azure App Insights |
|---|---|---|---|
| **Trace visualization** | Waterfall chart tốt | Waterfall chart tốt | Waterfall + Application Map |
| **Topology view** | Không built-in | Dependencies graph (limited) | **Application Map** — nodes + edges + metrics |
| **Query language** | Jaeger search UI | TraceQL | **KQL** — SQL-like, powerful aggregations |
| **Cross-signal join** | Không (traces only) | Partial (logs via Loki) | **operation_Id** joins traces + logs + exceptions |
| **AI-specific** | Không | Không | `customDimensions` cho LLM attributes |
| **Real-time** | Không | Không | **Live Metrics** |

**`operation_Id` là key join field cho tất cả tables.** Join `requests` với `dependencies` on `operation_Id` = thấy full chain từ incoming request đến tất cả downstream calls trong cùng trace.

**Exam trap:** `customDimensions["attribute.name"]` syntax — dùng bracket notation với string key, KHÔNG phải dot notation. Và phải `extend` trước khi dùng trong calculations.

---

## Checklist ghi nhớ cho AI-200

- [ ] Application Map: nodes = cloud roles · edges = calls between services
- [ ] Waterfall: root span top, children nested, horizontal timeline = timing
- [ ] All tables share `operation_Id` = OTel Trace ID
- [ ] requests table · dependencies table · traces table · exceptions table
- [ ] `join kind=inner (dependencies) on operation_Id` = link spans in same trace
- [ ] `customDimensions["attribute.name"]` = access span attributes in KQL
- [ ] `percentile(duration, 95)` = P95 latency analysis
- [ ] 4 AI patterns: embedding timeout, vector cold start, LLM rate limit, context overflow
- [ ] Live Metrics = real-time monitoring without full ingestion pipeline

---

*Module hoàn thành.*

---

[← Bài 4](./monitor-m1-bai4-export.md) · [🏠 Mục lục](../README.md) · [Assessment →](./monitor-m1-module-assessment.md)
