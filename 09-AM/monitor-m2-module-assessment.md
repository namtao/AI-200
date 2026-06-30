# Module Assessment — Analyze App Telemetry with Logs and Metrics

> Khoá: AI-200 · Azure Monitor — Analyze app telemetry with logs and metrics

---

## Câu 1

**Your AI application uses Application Insights with sampling enabled. You need to query the exceptions table to get an accurate count of total exceptions over the last 24 hours. Which KQL aggregation approach produces accurate results when sampling is active?**

- `sum(itemCount)` ✅
- `count()`
- `dcount(type)`

**Đáp án: `sum(itemCount)`**

Khi sampling active, App Insights giữ representative subset và assign `itemCount` cho mỗi sampled record = số actual events nó đại diện.

```kql
exceptions
| where timestamp > ago(24h)
| summarize exceptionCount = sum(itemCount) by bin(timestamp, 1h), type
```

- `count()` = số sampled records → **underrepresents** true volume
- `sum(itemCount)` = adjusted cho sampling → true volume

Tại sao các đáp án còn lại sai:
- **`count()`:** Chỉ count số rows trong result set. Nếu sampling rate 10%, 100 actual exceptions → 10 sampled records → `count()` trả về 10, không phải 100. Wrong.
- **`dcount(type)`:** Đây là probabilistic distinct count của exception types (bao nhiêu loại exceptions khác nhau). Không phải total exception count. Sai hoàn toàn.

---

## Câu 2

**A developer needs to trace the full chain of events from a single user request as it flows through four services in a document processing pipeline. Which Application Insights column links all telemetry items from that single distributed request?**

- `cloud_RoleName`
- `operation_Id` ✅
- `customDimensions`

**Đáp án: `operation_Id`**

`operation_Id` = OpenTelemetry **Trace ID**. Tất cả spans (requests, dependencies, exceptions, traces) từ cùng distributed request share cùng `operation_Id`. Đây là cách join telemetry qua tables:

```kql
exceptions
| join kind=inner (
    requests | project requestName = name, operation_Id, requestRoleName = cloud_RoleName
) on operation_Id
```

Tại sao các đáp án còn lại sai:
- **`cloud_RoleName`:** Identifies **which service** generated telemetry (vd. "embedding-service"). Không link individual request qua services — tất cả requests từ embedding-service share cùng cloud_RoleName.
- **`customDimensions`:** Property bag của custom attributes từ `set_attribute()`. Không phải built-in correlation ID — không automatically link requests qua services.

---

## Câu 3

**During an incident, a team member confirms elevated error rates on a dashboard and needs to investigate the root cause by dynamically filtering telemetry by service, time window, and error type. Which Azure Monitor tool is designed for this interactive investigation?**

- Azure dashboards
- Azure Monitor metric alerts
- Azure Monitor Workbooks ✅

**Đáp án: Azure Monitor Workbooks**

Workbooks = interactive analysis. Parameters cho phép users:
- Chọn time range → tất cả queries update tự động
- Filter by service name dropdown
- Filter by error type
- Drill-down từ summary grid vào specific transactions

Dashboard xác nhận "elevated error rates" (at-a-glance). Workbook answers "why".

Tại sao các đáp án còn lại sai:
- **Azure dashboards:** Tiles static, auto-refresh nhưng không interactive. Không có dropdown filters, không có drill-down. "Confirms elevated error rates" = dashboard job done. Investigation needs Workbooks.
- **Azure Monitor metric alerts:** Alerts là proactive notifications khi thresholds exceeded. Không phải investigation tool. Alert đã fired → bây giờ cần investigate → Workbooks.

---

## Câu 4

**A developer creates a log search alert with a KQL query that counts failed requests per service and returns rows only for services exceeding ten failures. The threshold is set to greater than zero table rows. What behavior does this configuration produce?**

- The alert fires whenever any service has more than 10 failures in the evaluation window. ✅
- The alert fires once for each individual failed request detected in the evaluation window.
- The alert fires only when all monitored services have more than 10 failures simultaneously.

**Đáp án: Alert fires khi bất kỳ service nào có > 10 failures**

```kql
requests
| where success == false
| summarize failedCount = count() by cloud_RoleName
| where failedCount > 10    -- Only return rows for services exceeding threshold
```

Config: threshold = `> 0 rows`

Logic: Query return rows chỉ cho services với failedCount > 10. Nếu **bất kỳ row nào** returned → threshold (> 0 rows) satisfied → alert fires.

Tại sao các đáp án còn lại sai:
- **Fires once per individual failed request:** Không. Query aggregates rồi filter — không iterate per request. Nếu 50 requests fail cho 1 service, query trả về 1 row (that service), alert fires 1 lần.
- **Only when ALL services exceed 10:** Không. `where failedCount > 10` filter per service independently. Nếu chỉ "embedding-service" có 15 failures → 1 row returned → threshold met → alert fires. Không cần tất cả services.

---

## Câu 5

**A developer monitors an AI pipeline where average response times remain within acceptable limits, but users report occasional slow responses. Which KQL function reveals the response time experienced by the slowest five percent of requests?**

- `avg()`
- `percentile(duration, 95)` ✅
- `max(duration)`

**Đáp án: `percentile(duration, 95)`**

P95 = response time below which **95% of requests fall**. Ngược lại: 5% của requests có duration **lớn hơn** P95 value.

```kql
requests
| summarize p95Duration = percentile(duration, 95) by cloud_RoleName
| where p95Duration > 3000
```

Vd: avg = 200ms (fine), P95 = 3000ms → 5% users đợi 3+ giây. Average che khuất vấn đề.

Tại sao các đáp án còn lại sai:
- **`avg()`:** Đây chính là vấn đề — average bị mask bởi majority fast requests. 95% complete 100ms, 5% complete 5s → avg ≈ 350ms (appears fine). `avg()` không reveal slowest 5%.
- **`max(duration)`:** Trả về worst case của tất cả requests — có thể là outlier bất thường, không represent systematic issue affecting 5% users. P95 là industry standard metric cho user experience, không phải max.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Accurate count khi sampling | `sum(itemCount)` |
| 2 | Column links telemetry qua services | `operation_Id` |
| 3 | Interactive incident investigation | Azure Monitor Workbooks |
| 4 | Alert behavior: > 0 rows threshold | Fires khi bất kỳ service nào > 10 failures |
| 5 | Slowest 5% requests | `percentile(duration, 95)` |

---

*Module hoàn thành.*

---

[← Bài 4](./monitor-m2-bai4-alerts.md) · [🏠 Mục lục](../README.md) · [Tổng hợp →](./monitor-summary.md)
