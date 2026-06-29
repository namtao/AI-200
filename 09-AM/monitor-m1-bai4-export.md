# Bài 4 — Export Telemetry to Azure Monitor

> Khoá: AI-200 · Azure Monitor — Instrument an app with OpenTelemetry

---

## Export Pipeline

```
App code → OTel SDK collects → Azure Monitor exporter serializes
         → Application Insights ingestion endpoint (via connection string)
```

**Direct export** (default) — app gửi thẳng đến App Insights, không cần intermediary.
**OTel Collector** (alternative) — separate process, centralized sampling/transformation/multi-backend. Phức tạp hơn.

---

## Signal → App Insights Table Mapping

| Signal | App Insights table |
|---|---|
| Server/Consumer spans | **requests** |
| Client/Internal/Producer spans | **dependencies** |
| Python logging / OTel Logs | **traces** |
| Exceptions | **exceptions** |
| Custom metrics (OTel Metrics API) | **customMetrics** |

---

## Sampling

Reduce telemetry volume → control ingestion costs. **Applies to traces only** (metrics never sampled).

### Fixed-percentage sampling

```python
configure_azure_monitor(
    sampling_ratio=0.1,    # 10% of traces
)
```

### Rate-limited sampling

```python
configure_azure_monitor(
    traces_per_second=1.5,    # ~1.5 traces/sec
)
```

### Environment variables (override code)

```bash
# Fixed percentage
export OTEL_TRACES_SAMPLER="microsoft.fixed_percentage"
export OTEL_TRACES_SAMPLER_ARG=0.1

# Rate-limited
export OTEL_TRACES_SAMPLER="microsoft.rate_limited"
export OTEL_TRACES_SAMPLER_ARG=1.5
```

**Env vars take precedence** over code config.

**Default:** `RateLimitedSampler` nếu không configure.

**Logs in unsampled traces:** Dropped by default (opt out possible).

**Good starting point:** 5% (0.05). Adjust based on Failures/Performance pane accuracy.

---

## Offline Storage và Retries

Exporter caches telemetry locally khi mất connectivity → retry đến **48 giờ**.

Default path: `<tempdir>/Microsoft-AzureMonitor-<hash>/opentelemetry-python-<instrumentation-key>`

```python
configure_azure_monitor(
    storage_directory="/var/telemetry/rag-pipeline",    # Custom path
)
```

`disable_offline_storage=True` → tắt (không khuyến nghị production).

---

## Verify Telemetry Flow

### App Insights Portal

Sau khi run app + generate traffic: kiểm tra server requests và response times trên overview pane.

### Live Metrics

Real-time, minimal delay. Enabled by default. Tốt cho verify custom spans trước production.

### KQL Query

```kql
requests
| where timestamp > ago(1h)
| project timestamp, name, duration, success, cloud_RoleName
| order by timestamp desc
| take 20
```

Thấy kết quả → telemetry pipeline working end-to-end. Thấy requests từ nhiều services → context propagation working.

---

## Bản chất bài này là gì?

**Một câu:** Export pipeline quyết định data đi đến đâu và bao nhiêu — sampling là lever quan trọng nhất để balance observability coverage với ingestion cost.

### So sánh với Prometheus/Grafana và AWS CloudWatch

| | Prometheus | AWS CloudWatch | Azure Monitor OTel |
|---|---|---|---|
| **Export model** | Pull (server scrapes) | Push (SDK/agent) | **Push** (direct to ingestion endpoint) |
| **Intermediary** | Cần scrape endpoint | CloudWatch agent optional | OTel Collector optional |
| **Sampling** | Không (all metrics) | Sampling configurable | `sampling_ratio` hoặc `traces_per_second` |
| **Offline resilience** | Không built-in | CloudWatch agent buffers | **48h local cache + retry** |
| **Metrics never sampled** | N/A — all metrics | N/A — all metrics | **Đúng — chỉ traces bị sample** |
| **Default sampler** | N/A | N/A | **RateLimitedSampler** |

**Sampling chỉ áp dụng cho traces, KHÔNG áp dụng cho metrics.** Metrics luôn được gửi đầy đủ. Đây là thiết kế quan trọng vì metrics cần accurate aggregation.

**Exam trap:** Env vars override code config cho sampling (`OTEL_TRACES_SAMPLER`, `OTEL_TRACES_SAMPLER_ARG`). Đây là ngược với connection string precedence intuition — nhớ rằng sampler env vars win over code.

---

## Checklist ghi nhớ cho AI-200

- [ ] Direct export = default (no Collector needed)
- [ ] Server/Consumer → requests · Client/Internal/Producer → dependencies
- [ ] Python logging → traces · Exceptions → exceptions
- [ ] Sampling = traces only (metrics never sampled)
- [ ] `sampling_ratio=0.1` = 10% · `traces_per_second=1.5`
- [ ] Env vars override code config cho sampling
- [ ] Default sampler: **RateLimitedSampler**
- [ ] Starting point: **5% (0.05)**
- [ ] Offline storage: cache locally, retry **48 giờ**
- [ ] Live Metrics: real-time verify, enabled by default

---

*Bài tiếp theo: Bài 5 — Debug distributed flows with trace data*
