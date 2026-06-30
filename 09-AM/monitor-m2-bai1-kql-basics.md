# Bài 1 — Write Basic KQL Queries

> Khoá: AI-200 · Azure Monitor — Analyze app telemetry with logs and metrics

---

## KQL và Azure Monitor Logs

**Pipe model:** Query bắt đầu với table name → flow qua operators (`|`). Mỗi operator transform output của step trước.

**Query scoping:**
- Từ **Application Insights resource** → dùng table names: `requests`, `exceptions`, `dependencies`
- Từ **Log Analytics workspace** → dùng: `AppRequests`, `AppExceptions`, `AppDependencies`

---

## Application Insights Tables

| Table | Stores |
|---|---|
| **requests** | Incoming HTTP requests (duration, response code, success) |
| **dependencies** | Outgoing calls (databases, APIs, model endpoints) |
| **exceptions** | Errors + stack traces |
| **traces** | Application log messages (ILogger, Serilog...) |
| **customEvents** | Custom business events (document classified, moderation flagged) |
| **customMetrics** | Custom numeric measurements (queue depth, model confidence) |
| **performanceCounters** | System-level metrics (CPU, memory, I/O) |

**Shared columns:**
- `timestamp` — khi event xảy ra
- `operation_Id` — links **all telemetry từ same distributed trace**
- `cloud_RoleName` — service nào generated telemetry
- `customDimensions` — key-value pairs từ `span.set_attribute()`
- `itemCount` — actual events per sampled record

---

## Core Operators

### where — Filter rows

```kql
requests
| where timestamp > ago(1h)
| where success == false
```

String operators: `has` (term index, fast) · `contains` (substring, slower) · `startswith`

### project — Select columns

```kql
| project timestamp, name, resultCode, duration, cloud_RoleName
```

### take / top

```kql
| take 20                    -- Arbitrary N rows
| top 20 by duration desc   -- N rows sorted
```

**Combined example:**

```kql
requests
| where timestamp > ago(1h)
| where success == false
| project timestamp, name, resultCode, duration, cloud_RoleName
| top 20 by duration desc
```

---

## Aggregate với summarize

```kql
requests
| where timestamp > ago(24h)
| summarize requestCount = count(), avgDuration = avg(duration)
    by bin(timestamp, 1h), cloud_RoleName
```

**Aggregation functions:**
- `count()` — row count
- `avg()`, `sum()`, `min()`, `max()`
- `dcount()` — distinct values (probabilistic, scale)
- `count_distinct()` — exact distinct (resource-intensive)
- `percentile(duration, 95)` — P95 latency

**`bin(timestamp, 1h)`** — group timestamps vào fixed intervals. Cần thiết cho time series.

---

## Visualize với render

```kql
requests
| where timestamp > ago(24h)
| summarize requestCount = count() by bin(timestamp, 1h), cloud_RoleName
| render timechart
```

Chart types: `timechart` · `barchart` · `piechart` · `columnchart` · `areachart`

`timechart` cần datetime column + numeric columns. Phổ biến nhất cho telemetry.

---

## Bản chất bài này là gì?

**Một câu:** KQL là SQL nhưng với pipe model và time-series focus — hiểu pipe model và 5 core operators (`where`, `project`, `summarize`, `bin`, `render`) là đủ để query bất kỳ telemetry nào.

### So sánh với PromQL (Prometheus) và Splunk SPL

| | PromQL | Splunk SPL | KQL |
|---|---|---|---|
| **Model** | Functional (instant/range vectors) | Pipe | **Pipe** (`table \| operator \| operator`) |
| **Strengths** | Metrics, alerting | Log search | **Logs + metrics + traces (unified)** |
| **Time grouping** | `[5m]` rate windows | `timechart span=1h` | **`bin(timestamp, 1h)`** |
| **Aggregation** | `sum by()`, `avg by()` | `stats avg() by` | **`summarize avg() by`** |
| **String search** | `=~"pattern"` regex | `*keyword*` wildcards | `has` (fast) · `contains` (slow) |
| **Multi-table join** | Không native | `append`, `join` | **`join kind=inner on field`** |
| **Visualization** | Grafana external | Built-in | **`\| render timechart`** built-in |

**`has` nhanh hơn `contains`** vì `has` dùng term index. Cho log queries với keywords → luôn dùng `has`.

**Exam trap:** App Insights dùng lowercase table names (`requests`, `exceptions`). Log Analytics workspace dùng `AppRequests`, `AppExceptions`. Nếu query từ wrong scope → không có data, không có error message rõ.

---

## Checklist ghi nhớ cho AI-200

- [ ] KQL = pipe model: table → `|` operator → `|` operator
- [ ] App Insights tables: requests, dependencies, exceptions, traces, customEvents
- [ ] `operation_Id` = links **tất cả telemetry từ same distributed trace**
- [ ] `cloud_RoleName` = identifies service
- [ ] `customDimensions` = span attributes từ `set_attribute()`
- [ ] `where`, `project`, `top`, `summarize`, `render`
- [ ] `bin(timestamp, 1h)` = group timestamps cho time series
- [ ] `percentile(duration, 95)` = P95 latency
- [ ] `render timechart` = line chart theo time

---

[← Assessment](./monitor-m1-module-assessment.md) · [🏠 Mục lục](../README.md) · [Bài 2 →](./monitor-m2-bai2-errors-performance.md)
