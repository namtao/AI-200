# Bài 2 — Explore Logs for Errors and Performance

> Khoá: AI-200 · Azure Monitor — Analyze app telemetry with logs and metrics

---

## Investigate Errors với KQL

### Exception rates over time

```kql
exceptions
| where timestamp > ago(24h)
| summarize exceptionCount = sum(itemCount) by bin(timestamp, 1h), type
| render timechart
```

**`sum(itemCount)` thay vì `count()`** khi sampling enabled:
- `count()` = số sampled records (underrepresents thực tế)
- `sum(itemCount)` = adjusted cho sampling = true volume

### Top exception types + operations

```kql
exceptions
| where timestamp > ago(24h)
| summarize exceptionCount = sum(itemCount) by type, operation_Name
| top 10 by exceptionCount desc
```

---

## Correlate Failures Across Services

Join trên `operation_Id` để see full request chain:

```kql
exceptions
| where timestamp > ago(24h)
| join kind=inner (
    requests
    | project requestName = name, requestDuration = duration, operation_Id, requestRoleName = cloud_RoleName
) on operation_Id
| project timestamp, exceptionType = type, exceptionMessage = outerMessage,
    requestName, requestDuration, cloud_RoleName = requestRoleName
| top 20 by timestamp desc
```

**Rename columns trước join** để tránh duplicate column names (KQL append suffix `1` cho duplicates).

---

## Analyze Dependency Latency

### Latency percentiles by target

```kql
dependencies
| where timestamp > ago(24h)
| summarize avg(duration), percentiles(duration, 50, 95, 99) by target, type
| order by percentile_duration_95 desc
```

P95/P99 reveal outliers mà average che khuất. Vd: avg 200ms nhưng P95 = 2s → problematic.

### Failed dependencies

```kql
dependencies
| where timestamp > ago(24h)
| where success == false
| summarize failureCount = count() by target, resultCode, cloud_RoleName
| order by failureCount desc
```

**resultCode diagnostics:**
- **HTTP 429** → rate limiting (adjust throughput, retry policies)
- **HTTP 500** → server-side failure in dependency
- **No code (timeout)** → network issue or overloaded service

---

## Built-in Views vs KQL

| | Failures View | Performance View | KQL |
|---|---|---|---|
| Location | Investigate > Failures | Investigate > Performance | Logs menu |
| Best for | Quick starting point, transaction drill-down | Response time distribution | Custom aggregations, joins, dashboards |

**Failures view:** Groups by operation + top 3 response codes/exceptions/dependencies → drill into transaction timeline.

**Performance view:** Ranked by duration/count → duration distribution chart. Good for bimodal patterns.

**Workflow:** Start với built-in views → switch sang KQL cho deeper analysis.

---

## Bản chất bài này là gì?

**Một câu:** `sum(itemCount)` thay vì `count()` khi sampling enabled, `join on operation_Id` để correlate failures qua services, và `percentile(duration, 95)` để tìm outliers mà average che khuất.

### So sánh với Grafana + Loki và Datadog Log Management

| | Grafana + Loki | Datadog Logs | Azure Monitor KQL |
|---|---|---|---|
| **Sampling compensation** | Không built-in | `@count` field | **`sum(itemCount)`** |
| **Cross-service correlation** | Trace ID label matching | `@trace_id` | **`join kind=inner on operation_Id`** |
| **Percentile query** | `histogram_quantile(0.95, ...)` | `p95(duration)` | **`percentile(duration, 95)`** |
| **Rate limiting detection** | Alert on 429 log line | Facet filter | `where resultCode == "429"` |
| **Column rename pre-join** | N/A | N/A | **`project renamed = col` trước join** |
| **Built-in failure views** | Không (external dashboards) | Log Explorer | **Investigate > Failures** portal view |

**`sum(itemCount)` là CRITICAL khi sampling enabled.** Sampling rate 10% → `count()` = 10, `sum(itemCount)` = 100. Exam luôn test điểm này.

**Exam trap:** Khi join 2 tables trong KQL mà có cùng column name (`name`, `duration`), KQL append suffix `1` cho table bên phải — `name1`, `duration1`. Luôn rename trước join để tránh confusion: `project requestName = name, operation_Id`.

---

## Checklist ghi nhớ cho AI-200

- [ ] **`sum(itemCount)`** = accurate count khi sampling enabled (không phải `count()`)
- [ ] `join kind=inner (table) on operation_Id` = correlate spans trong same trace
- [ ] Rename columns inside subquery trước join (tránh suffix `1`)
- [ ] `percentiles(duration, 50, 95, 99)` = multi-percentile trong one summarize
- [ ] HTTP 429 → rate limiting · HTTP 500 → server error · no code → timeout
- [ ] Failures view (Investigate > Failures) · Performance view (Investigate > Performance)
- [ ] Built-in views → quick start · KQL → custom analysis

---

*Bài tiếp theo: Bài 3 — Build dashboards + Create workbooks*
