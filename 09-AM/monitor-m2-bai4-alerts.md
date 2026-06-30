# Bài 4 — Set Alerts for App Failures and Anomalies

> Khoá: AI-200 · Azure Monitor — Analyze app telemetry with logs and metrics

---

## Alert Rule Structure — 4 Components

| Component | Description |
|---|---|
| **Scope** | Azure resource monitored (App Insights, Log Analytics workspace) |
| **Condition** | Signal + threshold logic (khi nào fire) |
| **Action group** | Notifications + automated actions khi fire |
| **Severity** | 0 (Critical) → 4 (Verbose) |

**2 alert types:**
- **Metric alerts** — evaluate resource metrics at regular intervals. Simple thresholds (CPU > 80%).
- **Log search alerts** — run KQL query at frequency, evaluate results. **Flexible, common cho AI apps** (joins, percentiles, custom dimensions).

---

## Severity Levels

| Level | Name | Dùng khi |
|---|---|---|
| 0 | **Critical** | Production outage, immediate response |
| 1 | Error | Significant problems affecting functionality |
| 2 | Warning | Potential issues, needs attention |
| 3 | Informational | Notable events, no immediate action |
| 4 | Verbose | Diagnostic information |

---

## Log Search Alert Rules

### Config options

- **Evaluation frequency** — how often query runs (1 min → 24h)
- **Aggregation granularity (window size)** — time window per evaluation
- **Threshold condition** — rows returned, or numeric value
- **Violations before fire** — prevent single transient spikes

### High failure rate detection

```kql
requests
| where success == false
| summarize failedCount = count() by cloud_RoleName
| where failedCount > 10
```

Config: `aggregation granularity = 5 min`, `evaluation frequency = 5 min`, `threshold > 0 rows`

→ Alert fires nếu **any row returned** = bất kỳ service nào > 10 failures trong 5 phút.

### Latency SLA alert

```kql
requests
| summarize p95Duration = percentile(duration, 95) by cloud_RoleName
| where p95Duration > 3000
```

**Percentile-based > average-based** cho latency alerts: Average OK nhưng P95 > 3s = 5% users bị slow.

---

## Action Groups

Reusable collection của notifications + automated actions. Attach to multiple alert rules. Up to 5 per alert rule.

### Notification types

- **Email** — alert summary + link to Azure portal
- **SMS** — short message (urgent alerts outside hours)
- **Azure mobile app push** — for Azure mobile app
- **Voice call** — automated call (highest severity)

### Automated actions

- **Azure Function** — remediation logic
- **Logic App** — create ticket in external system
- **Webhook** — third-party incident management
- **Automation Runbook** — scripted recovery
- **Event Hub** — stream alert data

**Design:** Separate action groups cho different severities:
- Critical: SMS + voice + webhook (incident management)
- Warning: email only (team distribution)

→ Avoid notification fatigue. **Test action groups** before production.

---

## Smart Detection

Machine learning detects unusual patterns **without manual thresholds**. Learns normal behavior over baseline period.

**Failure anomaly detection:** Monitors failed request rate vs historical baseline. Provides cluster analysis: affected users, correlated exceptions, related dependencies.

**Performance anomaly detection:** Gradual degradation patterns (gradual slowdowns không trigger fixed threshold).

### Smart detection vs Manual alerts

| | Smart detection | Manual log search alerts |
|---|---|---|
| Use for | Unexpected, novel problems | Known SLA thresholds, business rules |
| Threshold | ML-based, auto-learned | Manually configured |
| Example | New exception type appears suddenly | P95 must stay under 3s |

**Best practice: Use BOTH together** — manual for known boundaries + smart detection for novel problems.

---

## Bản chất bài này là gì?

**Một câu:** Log search alerts cho AI apps vì KQL queries (joins, percentiles, custom dimensions) — kết hợp với Smart Detection để cover cả known SLAs lẫn unknown anomalies.

### So sánh với Prometheus Alertmanager và Datadog Monitors

| | Prometheus + Alertmanager | Datadog Monitors | Azure Monitor Alerts |
|---|---|---|---|
| **Alert types** | PromQL rules | Metric, Log, APM monitors | **Metric alerts + Log search alerts** |
| **Query language** | PromQL | Datadog query language | **KQL** (log search) |
| **Severity levels** | Labels (custom) | P1-P5 | **0 Critical → 4 Verbose** |
| **Action routing** | Alertmanager routes | Notification rules | **Action groups** (reusable) |
| **ML anomaly** | Không built-in | Anomaly detection add-on | **Smart Detection** (built-in) |
| **Gradual degradation** | Cần recording rules | Anomaly threshold | **Smart Detection tự học** |
| **Action types** | Webhook | Webhook + integrations | Email, SMS, Function, Logic App, **Runbook** |
| **AI-specific alerts** | Cần custom metrics | Custom monitors | KQL trên `customDimensions` |

**Use BOTH smart detection AND manual alerts.** Smart detection catches novel problems (không biết threshold trước). Manual log search alerts enforce known SLAs (P95 < 3s). Đây là best practice, không phải either/or.

**Exam trap:** Log search alert fires khi `> 0 rows` returned — KQL filter trong query (`where failedCount > 10`) rồi alert threshold `> 0 rows` = alert fires nếu ANY service exceeds threshold. Không cần tất cả services vi phạm cùng lúc.

---

## Checklist ghi nhớ cho AI-200

- [ ] 4 components: Scope, Condition, **Action group**, Severity
- [ ] Severity: 0 Critical → 4 Verbose
- [ ] Metric alerts = simple thresholds · Log search alerts = **flexible KQL, AI apps**
- [ ] Log search: evaluation frequency, aggregation granularity (window size)
- [ ] `threshold > 0 rows` = fires nếu query returns bất kỳ row nào
- [ ] Percentile-based > average-based cho latency alerts
- [ ] Action group: reusable, up to 5 per rule
- [ ] Smart detection: ML-based, learns baseline, catch gradual degradation
- [ ] Use both: smart detection (novel) + manual alerts (known SLAs)

---

*Module Assessment tiếp theo*

---

[← Bài 3](./monitor-m2-bai3-dashboards-workbooks.md) · [🏠 Mục lục](../README.md) · [Assessment →](./monitor-m2-module-assessment.md)
