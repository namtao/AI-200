# Tổng hợp — Azure Monitor và OpenTelemetry

> Khoá: AI-200 · Module 09-AM

---

## Bức tranh tổng thể

```
┌─────────────────────────────────────────────────────────────────┐
│  AI App (RAG Pipeline, Microservices)                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ API Gateway  │  │  Embedding   │  │  LLM Service │          │
│  │ (SERVER span)│→ │  Service     │→ │  (CLIENT span│          │
│  └──────────────┘  │ (CLIENT span)│  └──────────────┘          │
│                    └──────────────┘                             │
│         traceparent header propagates trace_id across all       │
└─────────────────────────────────────────────────────────────────┘
                              │
                    OTel SDK collects
                              │
              ┌───────────────▼────────────────┐
              │  Azure Monitor Exporter        │
              │  (azure-monitor-opentelemetry) │
              └───────────────┬────────────────┘
                              │ HTTPS
              ┌───────────────▼────────────────┐
              │  Application Insights          │
              │  ┌──────────┬──────────────┐   │
              │  │requests  │ dependencies │   │
              │  ├──────────┼──────────────┤   │
              │  │exceptions│   traces     │   │
              │  ├──────────┼──────────────┤   │
              │  │customMetrics │ customEvents│  │
              │  └──────────┴──────────────┘   │
              └────────────────────────────────┘
                              │
              ┌───────────────▼────────────────┐
              │  Analysis & Alerting           │
              │  KQL Queries · Dashboards      │
              │  Workbooks · Smart Detection   │
              └────────────────────────────────┘
```

---

## Module 1 — OpenTelemetry Instrumentation

### M1 Overview

| Bài | Chủ đề | Key takeaway |
|---|---|---|
| Bài 1 | OTel Concepts | 3 pillars, W3C traceparent, OTel ↔ App Insights mapping |
| Bài 2 | Setup | `pip install azure-monitor-opentelemetry`, `configure_azure_monitor()`, cloud role name |
| Bài 3 | Spans & Traces | Custom spans, SpanKind → table mapping, nested spans |
| Bài 4 | Export | Sampling (traces only), offline storage 48h, env vars override |
| Bài 5 | Debug Traces | Application Map, waterfall chart, `operation_Id` join, AI patterns |

### M1 Core Concepts

**3 Pillars of Observability:**
- **Traces** → Where delays/errors occur (distributed request path)
- **Metrics** → When something changed (aggregate numbers over time)
- **Logs** → Why something happened (timestamped event records)

**OTel Architecture:**
```
APIs → SDKs → Instrumentation Libraries → Exporters → Backend
```

**Azure Monitor Distro** = SDK + Azure exporter + auto-instrumentation libraries bundled.

**Context Propagation — W3C TraceContext:**
```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             {version}-{trace-id-32hex}-{parent-span-id-16hex}-{flags}
```

**SpanKind → App Insights Table:**

| SpanKind | App Insights table |
|---|---|
| `SERVER` | **requests** |
| `CONSUMER` | **requests** |
| `CLIENT` | **dependencies** |
| `INTERNAL` (default) | **dependencies** |
| `PRODUCER` | **dependencies** |

**OTel ↔ App Insights field mapping:**

| OTel | App Insights |
|---|---|
| Trace ID | `operation_Id` |
| Span attributes (`set_attribute()`) | `customDimensions` |
| `service.name` + `service.namespace` | Cloud role name |
| Server/Consumer span | `requests` table |
| Client/Internal/Producer span | `dependencies` table |
| Python logging | `traces` table |

---

## Module 2 — KQL và Azure Monitor Analysis

### M2 Overview

| Bài | Chủ đề | Key takeaway |
|---|---|---|
| Bài 1 | KQL Basics | Pipe model, core operators, `bin()`, `render` |
| Bài 2 | Errors & Performance | `sum(itemCount)`, joins, percentiles, resultCode diagnostics |
| Bài 3 | Dashboards & Workbooks | Static vs interactive, `{Param}` syntax, aggregation types |
| Bài 4 | Alerts | Log search alerts, action groups, Smart Detection |

### App Insights Tables Reference

| Table | Stores | Key columns |
|---|---|---|
| **requests** | Incoming HTTP (SERVER/CONSUMER spans) | `name`, `duration`, `success`, `resultCode` |
| **dependencies** | Outgoing calls (CLIENT/INTERNAL spans) | `name`, `target`, `type`, `duration`, `success` |
| **exceptions** | Errors + stack traces | `type`, `outerMessage`, `itemCount` |
| **traces** | Log messages | `message`, `severityLevel` |
| **customEvents** | Business events | `name`, `customDimensions` |
| **customMetrics** | Custom measurements | `name`, `value`, `customDimensions` |

**Shared columns across all tables:**
- `timestamp` — khi event xảy ra
- `operation_Id` — **links tất cả telemetry từ same distributed trace** (= OTel Trace ID)
- `cloud_RoleName` — service nào generated telemetry
- `customDimensions` — span attributes từ `set_attribute()`
- `itemCount` — actual events per sampled record (dùng với `sum()`)

---

## KQL Quick Reference

### Core Operators

```kql
-- Filter
| where timestamp > ago(1h)
| where success == false
| where cloud_RoleName == "embedding-service"

-- Select columns
| project timestamp, name, duration, cloud_RoleName

-- Sort & limit
| top 20 by duration desc
| take 20

-- Aggregate (always with by clause for grouping)
| summarize count(), avg(duration), percentile(duration, 95) by cloud_RoleName

-- Time bucket (required for timechart)
| summarize count() by bin(timestamp, 1h)

-- Visualize
| render timechart
| render barchart
| render piechart
```

### String Operators (performance matters)

```kql
| where message has "error"        -- Fast: uses term index
| where message contains "error"   -- Slow: substring scan
| where name startswith "Generate" -- Fast for prefix
| where name matches regex @"^Gen" -- Slowest
```

### Aggregation Functions

```kql
count()                           -- Row count
sum(itemCount)                    -- Sampling-aware count (use this!)
avg(duration)                     -- Average
percentile(duration, 95)          -- P95 latency
percentiles(duration, 50, 95, 99) -- Multi-percentile in one go
dcount(userId)                    -- Distinct count (probabilistic, fast)
count_distinct(userId)            -- Exact distinct (slow)
countif(success == false)         -- Conditional count
```

### Cross-Table Join Pattern

```kql
-- Template: join exceptions with their parent requests
exceptions
| where timestamp > ago(24h)
| join kind=inner (
    requests
    | project requestName = name,       -- Rename to avoid suffix collision
              requestDuration = duration,
              operation_Id,
              requestRoleName = cloud_RoleName
) on operation_Id
| project timestamp, exceptionType = type, exceptionMessage = outerMessage,
    requestName, requestDuration, cloud_RoleName = requestRoleName
| top 20 by timestamp desc
```

### Custom Dimensions Access

```kql
-- Access span attributes in KQL
dependencies
| where name == "GenerateEmbedding"
| extend tokenCount = toint(customDimensions["embedding.token_count"]),
         model = tostring(customDimensions["embedding.model"])
| summarize avg(duration), percentile(duration, 95), avg(tokenCount) by model
```

### AI-Service Query Patterns

```kql
-- 1. LLM latency percentiles
dependencies
| where name == "CallLlmApi"
| where timestamp > ago(24h)
| summarize p50 = percentile(duration, 50),
            p95 = percentile(duration, 95),
            p99 = percentile(duration, 99)
    by bin(timestamp, 1h)
| render timechart

-- 2. Embedding token usage by model
dependencies
| where name == "GenerateEmbedding"
| extend tokens = toint(customDimensions["embedding.token_count"]),
         model = tostring(customDimensions["embedding.model"])
| summarize totalTokens = sum(tokens), avgDuration = avg(duration),
            callCount = count() by model
| order by totalTokens desc

-- 3. Rate limiting detection (HTTP 429)
dependencies
| where timestamp > ago(1h)
| where success == false
| where resultCode == "429"
| summarize count() by target, cloud_RoleName
| order by count_ desc

-- 4. Slow requests with downstream cause
requests
| where duration > 5000
| join kind=inner (dependencies) on operation_Id
| project timestamp, requestName = name, requestDuration = duration,
    dependencyName = name1, dependencyDuration = duration1,
    dependencyTarget = target
| order by dependencyDuration desc

-- 5. Exception rate with sampling compensation
exceptions
| where timestamp > ago(24h)
| summarize exceptionCount = sum(itemCount) by bin(timestamp, 1h), type
| render timechart
```

### Alert KQL Templates

```kql
-- High failure rate (fires if any service > 10 failures in window)
requests
| where success == false
| summarize failedCount = count() by cloud_RoleName
| where failedCount > 10

-- SLA breach (P95 > 3s)
requests
| summarize p95Duration = percentile(duration, 95) by cloud_RoleName
| where p95Duration > 3000

-- Dependency failures by type
dependencies
| where success == false
| summarize failureCount = count() by target, resultCode, cloud_RoleName
| where failureCount > 5
```

---

## OTel Instrumentation Patterns cho AI Service

### Minimal Setup

```python
import os
from azure.monitor.opentelemetry import configure_azure_monitor
from opentelemetry import trace
from opentelemetry.sdk.resources import Resource
from opentelemetry.trace import SpanKind, Status, StatusCode

# 1. Configure once at startup
configure_azure_monitor(
    resource=Resource.create({
        "service.name": "rag-pipeline",
        "service.namespace": "ai-200",
        "service.instance.id": os.environ.get("INSTANCE_ID", "local-1"),
    })
)

tracer = trace.get_tracer("rag-pipeline")
```

### RAG Pipeline Instrumentation

```python
def process_query(query: str) -> str:
    # Root span — incoming request
    with tracer.start_as_current_span("ProcessQuery", kind=SpanKind.SERVER) as root:
        root.set_attribute("query.length", len(query))
        root.set_attribute("query.hash", hash(query))

        # Embedding call — outgoing to embedding service
        with tracer.start_as_current_span("GenerateEmbedding", kind=SpanKind.CLIENT) as embed:
            embed.set_attribute("embedding.model", "text-embedding-ada-002")
            embedding = embedding_service.generate(query)
            embed.set_attribute("embedding.dimensions", len(embedding))
            embed.set_attribute("embedding.token_count", count_tokens(query))

        # Vector search — internal operation
        with tracer.start_as_current_span("VectorSearch") as search:  # default = INTERNAL
            search.set_attribute("search.index_name", "product-docs")
            search.set_attribute("search.top_k", 5)
            results = vector_db.search(embedding, top_k=5)
            search.set_attribute("search.result_count", len(results))

        # LLM call — outgoing to Azure OpenAI
        with tracer.start_as_current_span("CallLlm", kind=SpanKind.CLIENT) as llm:
            llm.set_attribute("llm.provider", "azure-openai")
            llm.set_attribute("llm.model", "gpt-4o")
            prompt = build_prompt(query, results)
            llm.set_attribute("llm.prompt_tokens", count_tokens(prompt))
            try:
                response = llm_client.complete(prompt)
                llm.set_attribute("llm.response_tokens", response.usage.completion_tokens)
                llm.set_status(Status(StatusCode.OK))
                return response.content
            except Exception as ex:
                llm.record_exception(ex)
                llm.set_status(Status(StatusCode.ERROR, str(ex)))
                raise
```

### Span Attribute Naming Convention

```python
# Good: namespaced keys
span.set_attribute("embedding.model", "text-embedding-ada-002")
span.set_attribute("embedding.token_count", 512)
span.set_attribute("llm.model", "gpt-4o")
span.set_attribute("llm.prompt_tokens", 1024)
span.set_attribute("llm.response_tokens", 256)
span.set_attribute("search.index_name", "docs")
span.set_attribute("search.top_k", 5)
span.set_attribute("search.result_count", 5)

# Bad: generic keys
span.set_attribute("value", 512)       # What value?
span.set_attribute("data", embedding)  # What data?
span.set_attribute("count", 5)         # Count of what?
```

### Sampling Configuration

```python
# Production: start with 5%
configure_azure_monitor(sampling_ratio=0.05)

# Or via environment (takes precedence over code)
# export OTEL_TRACES_SAMPLER="microsoft.fixed_percentage"
# export OTEL_TRACES_SAMPLER_ARG=0.05
```

---

## Exam Traps — 8+ điểm cần nhớ

### Trap 1: `operation_Id` vs `id`

- `operation_Id` = **Trace ID** (shared by all spans in same trace) → join key
- `id` = **Span ID** (unique per telemetry item) → NOT join key
- Exam hỏi "link telemetry qua services" → `operation_Id`

### Trap 2: `sum(itemCount)` vs `count()`

- Sampling enabled → `count()` = số sampled records (undercount)
- `sum(itemCount)` = true volume (sampling-adjusted)
- **Luôn dùng `sum(itemCount)` khi sampling active**
- Exception: khi hỏi "how many rows" → `count()`

### Trap 3: Sampling applies to traces only

- Metrics → **KHÔNG bao giờ sampled**
- Traces → sampled theo `sampling_ratio` hoặc `traces_per_second`
- Logs in unsampled traces → dropped by default

### Trap 4: Env var precedence cho sampling

- `OTEL_TRACES_SAMPLER` + `OTEL_TRACES_SAMPLER_ARG` → **override code config**
- `APPLICATIONINSIGHTS_CONNECTION_STRING` → recommended production (code > env var)
- 2 settings khác nhau có precedence rules ngược nhau — học theo từng cái

### Trap 5: SpanKind.INTERNAL là default

- Không set kind → INTERNAL → **dependencies table**
- Chỉ SERVER và CONSUMER đi vào **requests table**
- LLM call phải explicitly set `kind=SpanKind.CLIENT`

### Trap 6: Nested spans — KHÔNG pass references

- SDK tự động set parent từ Python context variables
- KHÔNG cần `tracer.start_as_current_span("child", parent=root_span)`
- Chỉ cần nest `with` blocks đúng cách

### Trap 7: KQL table names theo scope

- Query từ **Application Insights** → `requests`, `exceptions`, `dependencies` (lowercase)
- Query từ **Log Analytics workspace** → `AppRequests`, `AppExceptions`, `AppDependencies` (PascalCase)
- Wrong scope → empty results, no error

### Trap 8: Column rename trước join

- Join 2 tables với cùng column name (`name`, `duration`, `cloud_RoleName`) → KQL append `1`
- `dependencies` join → `name1`, `duration1`
- **Luôn rename trước join:** `project requestName = name, operation_Id`

### Trap 9: Dashboard vs Workbook cho investigation

- Dashboard = static tiles, auto-refresh → **"Is it healthy?"**
- Workbook = interactive parameters, drill-down → **"Why is it unhealthy?"**
- Exam scenario: "investigate root cause during incident" → **Workbooks**
- Exam scenario: "monitor ongoing health, wall screen" → **Dashboards**

### Trap 10: Alert threshold `> 0 rows`

- Log search alert: KQL returns rows → `> 0 rows` threshold → alert fires
- Pattern: filter per service (`where failedCount > 10`), then `threshold > 0`
- Fires if **ANY** service exceeds → không cần tất cả vi phạm cùng lúc

### Trap 11: Smart Detection vs Manual Alerts

- Smart Detection: ML-based, **không cần threshold**, catches gradual degradation
- Manual log search: **known SLA thresholds** (P95 < 3s, failure rate < 1%)
- Best practice: **BOTH** — không phải either/or
- Smart detection alone = miss known SLA violations
- Manual alone = miss novel/unexpected patterns

### Trap 12: `has` vs `contains` trong KQL

- `has "error"` → term index scan, **fast**
- `contains "error"` → substring scan, **slow**
- Cho log queries → luôn dùng `has` khi search exact word

---

## Assessment Summaries

### Module 1 Assessment — 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Context propagation standard | **W3C TraceContext** (`traceparent` header) — không phải B3/OpenCensus |
| 2 | Connection string production | **`APPLICATIONINSIGHTS_CONNECTION_STRING`** env var — không embed in code |
| 3 | Unhandled exception in span block | SDK **auto** records exception + sets ERROR status khi propagate ra ngoài |
| 4 | `SpanKind.CLIENT` → table | **`dependencies`** — không phải `requests` hay `traces` |
| 5 | OTel Trace ID ↔ App Insights | **`operation_Id`** — không phải `id` (span-level) |

**Key patterns từ M1 assessment:**
- W3C TraceContext là standard mới (IETF), thay thế B3 headers của Zipkin/OpenTracing
- Exception auto-recorded chỉ khi **propagate out** of `with` block — nếu caught bên trong thì không
- `traces` table = logs (Python logging), không phải spans

### Module 2 Assessment — 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Accurate count khi sampling | **`sum(itemCount)`** — `count()` underrepresents, `dcount()` là distinct count |
| 2 | Column links telemetry qua services | **`operation_Id`** — `cloud_RoleName` identifies service, `customDimensions` là attributes |
| 3 | Interactive incident investigation | **Azure Monitor Workbooks** — dashboards static, alerts proactive (không phải investigation) |
| 4 | Alert behavior: `> 0 rows` threshold | Fires khi **bất kỳ service nào** > 10 failures — không phải per request, không phải ALL services |
| 5 | Slowest 5% requests | **`percentile(duration, 95)`** — avg bị mask, max là outlier không representative |

**Key patterns từ M2 assessment:**
- `dcount()` = distinct count (bao nhiêu loại), không phải total count
- Workbooks là investigation tool sau khi dashboard confirms problem
- P95 = industry standard cho user experience SLAs — avg lies, max misleads

---

## Azure Monitor vs Prometheus/Grafana vs Datadog

### Comparison Matrix

| Dimension | Prometheus/Grafana | Datadog | Azure Monitor + OTel |
|---|---|---|---|
| **Instrumentation** | Client libs (vendor-specific) | Datadog agent + SDK | **OTel SDK (CNCF standard)** |
| **Vendor lock-in** | Locked to Prometheus format | **Highly locked** | OTel = portable, swap backends |
| **Setup complexity** | Cao (Prometheus + Grafana + AlertManager + exporters) | Trung bình (agent-based) | **Thấp** (1 pip install, 1 function call) |
| **Azure integration** | Cần Azure Monitor exporter | Cần Datadog Azure integration | **Native, no extra config** |
| **Query language** | PromQL (functional) | Datadog Query Language | **KQL (pipe model, SQL-like)** |
| **Trace correlation** | Exemplars (limited) | Trace ID linking | **`operation_Id` cross-table joins** |
| **AI pipeline** | Cần custom metrics + dashboards | tốt, có AI observability features | `customDimensions` cho LLM attributes |
| **Anomaly detection** | Cần external (Grafana ML plugin) | Anomaly detection add-on | **Smart Detection built-in** |
| **Cost model** | Self-hosted (infra cost) | Per-host + data ingestion | Pay per GB ingested |
| **Offline resilience** | Cần external queue | Agent buffers | **48h local cache + retry** |
| **Context propagation** | B3 headers (older) | x-datadog-trace-id | **W3C TraceContext (IETF standard)** |
| **Dashboard sharing** | Grafana org permissions | Datadog RBAC | Azure RBAC (dashboard + resource) |
| **Real-time monitoring** | Near real-time | Near real-time | **Live Metrics** (minimal latency) |

### When to choose what

**Azure Monitor + OTel khi:**
- App runs trên Azure (native integration, no extra config)
- Muốn portable instrumentation (có thể switch backend sau)
- Team không muốn manage Prometheus/Grafana infrastructure
- Cần intelligent anomaly detection without threshold tuning

**Prometheus/Grafana khi:**
- Multi-cloud hoặc on-premises
- Metrics-heavy workload (custom alerting, recording rules)
- Team đã quen với Kubernetes + Prometheus ecosystem
- Budget constraint (self-hosted)

**Datadog khi:**
- Multi-cloud, nhiều tech stacks khác nhau
- Muốn all-in-one với APM, Logs, Infrastructure in one platform
- Willing to pay premium cho UX và integrations
- Already standardized on Datadog across organization

### Technical Equivalents

| Azure Monitor | Prometheus/Grafana | Datadog |
|---|---|---|
| Application Insights | Jaeger + Prometheus + Grafana | APM + Log Management |
| `operation_Id` | Trace ID label | `@trace_id` |
| `customDimensions` | Prometheus labels | Log facets / span tags |
| `requests` table | HTTP request counter metrics | APM traces |
| `dependencies` table | Dependency span metrics | APM dependencies |
| KQL `summarize` | PromQL `sum by()` | `avg by:` |
| KQL `bin(timestamp, 1h)` | PromQL `[1h]` range | `rollup:1h` |
| Smart Detection | ML alert in Grafana (plugin) | Anomaly detection (add-on) |
| Workbooks | Grafana dashboards (variables) | Notebooks |
| Azure Dashboards | Grafana dashboards (static) | Dashboards |
| Action Groups | Alertmanager receivers | Notification policies |
| Live Metrics | Grafana real-time | Live Tail |

---

## Setup Checklist — Production AI App

```bash
# 1. Install
pip install azure-monitor-opentelemetry

# 2. Environment variables
export APPLICATIONINSIGHTS_CONNECTION_STRING="InstrumentationKey=...;IngestionEndpoint=..."
export OTEL_SERVICE_NAME="my-ai-service"
export OTEL_RESOURCE_ATTRIBUTES="service.namespace=ai-pipeline,service.instance.id=${HOSTNAME}"

# 3. Sampling (optional — default is RateLimitedSampler)
export OTEL_TRACES_SAMPLER="microsoft.fixed_percentage"
export OTEL_TRACES_SAMPLER_ARG=0.05    # 5% starting point
```

```python
# 4. Startup code
from azure.monitor.opentelemetry import configure_azure_monitor
configure_azure_monitor()    # Picks up env vars automatically

# 5. Tracer (once per component)
from opentelemetry import trace
tracer = trace.get_tracer("my-ai-service")
```

**Verify flow:**
```kql
-- Check if data arriving (run after generating some traffic)
requests
| where timestamp > ago(1h)
| project timestamp, name, duration, success, cloud_RoleName
| order by timestamp desc
| take 20
```

---

*Tổng hợp hoàn thành — M1 (5 bài) + M2 (4 bài) + 2 assessments.*

---

[← Assessment](./monitor-m2-module-assessment.md) · [🏠 Mục lục](../README.md) · [Mục lục →](../README.md)
