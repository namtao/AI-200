# Bài 3 — Dashboards và Workbooks

> Khoá: AI-200 · Azure Monitor — Analyze app telemetry with logs and metrics

---

## Azure Dashboards vs Workbooks

| | Dashboards | Workbooks |
|---|---|---|
| **Purpose** | At-a-glance operational awareness | Interactive investigation |
| **Interaction** | Static tiles, auto-refresh | Dynamic: parameters, filters, drill-down |
| **Question** | "Is the system healthy?" | "Why is it behaving this way?" |
| **Use case** | Ongoing monitoring, wall screens | Incident investigation, root cause analysis |

---

## Dashboards — Pin Metrics

**Metrics pane** (Monitoring > Metrics): Chọn metric → filter → split → pin to dashboard.

**Common metrics cho AI apps:**
- Server response time (avg + percentiles)
- Failed requests (rate + count)
- Dependency failures
- Server request rate (throughput req/s)
- Availability (uptime from availability tests)

**Aggregation types:**
- **Average** → typical behavior (response times, latency)
- **Sum** → total events over period
- **Max** → worst-case values (latency spikes)

---

## Dashboards — Pin Log Query Results

```kql
-- Failed requests by service → bar chart tile
requests
| where timestamp > ago(24h)
| where success == false
| summarize failedCount = count() by cloud_RoleName
| render barchart
```

```kql
-- P50/P95/P99 latency over time → time chart tile
requests
| where timestamp > ago(24h)
| summarize p50 = percentile(duration, 50),
    p95 = percentile(duration, 95),
    p99 = percentile(duration, 99)
    by bin(timestamp, 1h)
| render timechart
```

Tiles refresh on schedule. **Không phải real-time** — dùng Live Metrics cho incidents.

---

## Dashboard Design Best Practices

- Separate concerns: pipeline health dashboard ≠ business metrics dashboard
- Limit 5-10 tiles (focused, no clutter)
- Descriptive tile titles ("Classification Service P95 Latency", không phải "Query 3")
- Markdown tile ở top: application description, runbook links, on-call contacts

**Share:** Share > Publish → RBAC (user cần read access trên cả dashboard + App Insights resource).

---

## Workbooks — Interactive Analysis

**Step types:** Text blocks · Query steps · Metrics steps · Parameter steps

**Visualizations:** Grid (table) · Chart (line/bar/area) · Tiles (summary cards) · Map

### Parameter reference trong query

```kql
requests
| where timestamp > ago({TimeRange})    -- {TimeRange} = user-selected parameter
| summarize totalRequests = count(),
    failedRequests = countif(success == false)
    by cloud_RoleName
| extend successRate = round(100.0 * (totalRequests - failedRequests) / totalRequests, 2)
| project cloud_RoleName, totalRequests, failedRequests, successRate
```

`{ParameterName}` syntax → injected từ workbook parameter controls.

### Parameter types

| Type | Description |
|---|---|
| **Time range** | Standard dropdown: last 1h, 24h, 7d |
| **Dropdown** | Populated by KQL query hoặc static list |
| **Resource picker** | Select App Insights resource or workspace |

Multi-select dropdown → `| where cloud_RoleName in ({ServiceName})`

### Design pattern

1. Summary query (overview across all services)
2. Parameter filters (time range, service)
3. Linked grid → drill-down per service
4. Conditionally visible detail steps (show on demand)

---

## Bản chất bài này là gì?

**Một câu:** Dashboards trả lời "hệ thống có healthy không?" (static, always-on), Workbooks trả lời "tại sao nó unhealthy?" (interactive, investigation) — chọn đúng tool theo câu hỏi.

### So sánh với Grafana Dashboards và Datadog Dashboards

| | Grafana Dashboards | Datadog Dashboards | Azure Dashboards | Azure Workbooks |
|---|---|---|---|---|
| **Purpose** | Monitoring + alerting | Monitoring + alerting | **At-a-glance health** | **Interactive investigation** |
| **Interactivity** | Variables (limited) | Template variables | Static tiles | **Parameters, filters, drill-down** |
| **Real-time** | Near real-time (configurable) | Near real-time | Auto-refresh (schedule) | On-demand refresh |
| **Parameter syntax** | `$variable` | `{{variable}}` | N/A | **`{ParameterName}`** |
| **Multi-select** | `$variable:csv` | `{{variable}}` | N/A | `where col in ({Param})` |
| **Sharing** | Grafana org permissions | Datadog RBAC | **RBAC — reader on dashboard + resource** | RBAC similar |
| **Incident use** | Sửa dashboard mid-incident | Notebook feature | Not ideal | **Designed for this** |

**Dashboards tiles không real-time** — refresh on schedule. Dùng Live Metrics cho real-time monitoring trong incidents, không phải dashboard.

**Exam trap:** Workbook parameter injection syntax là `{ParameterName}` (curly braces), KHÔNG phải `$param` (Grafana style) hay `{{param}}` (Datadog style). Multi-select dùng `in ({Param})`.

---

## Checklist ghi nhớ cho AI-200

- [ ] Dashboards = static tiles, auto-refresh · Workbooks = interactive parameters
- [ ] Dashboard từ Metrics pane hoặc Logs query result
- [ ] Aggregation: Average (latency) · Sum (counts) · Max (worst-case)
- [ ] Tiles refresh on schedule (không real-time) → Live Metrics cho incidents
- [ ] Workbook parameters: `{TimeRange}`, `{ServiceName}` syntax
- [ ] Multi-select: `where cloud_RoleName in ({ServiceName})`
- [ ] Dashboard best practices: 5-10 tiles, descriptive titles, separate concerns

---

[← Bài 2](./monitor-m2-bai2-errors-performance.md) · [🏠 Mục lục](../README.md) · [Bài 4 →](./monitor-m2-bai4-alerts.md)
