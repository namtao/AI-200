# Bài 2 — Add the OpenTelemetry SDK to an Application

> Khoá: AI-200 · Azure Monitor — Instrument an app with OpenTelemetry

---

## 2 Instrumentation Approaches

| Approach | Description | Khi dùng |
|---|---|---|
| **Autoinstrumentation** | No code changes, config-based | App Service, Functions, VMs |
| **Manual (SDK)** | SDK embedded in code | **AI apps — custom spans, business metrics** |

**SDK-based preferred cho AI** vì cần capture business operations (embedding timing, LLM tokens, etc.)

---

## Install

```bash
pip install azure-monitor-opentelemetry
```

```python
from azure.monitor.opentelemetry import configure_azure_monitor

configure_azure_monitor()    # Initialize tracer, meter, logger providers
```

Minimal setup: 1 lần tại app startup.

---

## Automatic Data Collection

Không cần viết thêm code:
- **`requests` library** → outgoing HTTP calls (embedding API, LLM API) as client spans
- **`urllib` / `urllib3`** → outgoing HTTP
- **Flask / Django / FastAPI** → incoming requests as server spans
- **`psycopg2`** → PostgreSQL queries as dependency spans
- **Azure SDK** → calls to Azure services
- **Python `logging` module** → flows into App Insights automatically

---

## Connection String

Unique per Application Insights resource. 3 cách config (precedence: **code > env var**):

```bash
# Recommended cho production (env var)
export APPLICATIONINSIGHTS_CONNECTION_STRING="InstrumentationKey=00000000...;IngestionEndpoint=https://eastus-0.in.applicationinsights.azure.com/"
```

```python
# Code-based (không khuyến nghị production — embeds credentials)
configure_azure_monitor(
    connection_string="InstrumentationKey=00000000...;IngestionEndpoint=..."
)
```

---

## Cloud Role Name

Mỗi service cần **unique cloud role name** để appear as separate node trên Application Map.

Derived từ `service.namespace` + `service.name` resource attributes.

```python
from azure.monitor.opentelemetry import configure_azure_monitor
from opentelemetry.sdk.resources import Resource

configure_azure_monitor(
    resource=Resource.create({
        "service.name": "embedding-service",
        "service.namespace": "rag-pipeline",        # Groups services logically
        "service.instance.id": "embedding-instance-1",
    })
)
# Cloud role name: "rag-pipeline.embedding-service"
```

```bash
# Via environment variables
export OTEL_SERVICE_NAME="embedding-service"
export OTEL_RESOURCE_ATTRIBUTES="service.namespace=rag-pipeline,service.instance.id=embedding-instance-1"
```

Không có unique names → tất cả services appear as 1 node → Application Map vô dụng.

---

## Bản chất bài này là gì?

**Một câu:** Một package, một function call tại startup, một env var cho connection string — Azure Monitor Distro làm cho việc instrument AI app trở nên tối giản nhất có thể.

### So sánh với Prometheus/Grafana và Datadog

| | Prometheus/Grafana | Datadog | Azure Monitor Distro |
|---|---|---|---|
| **Setup complexity** | Cao (Prometheus server + Grafana + exporters) | Trung bình (agent install + SDK) | **Thấp** (`pip install` + 1 function call) |
| **Auto-instrumentation** | Cần configure scrapers thủ công | Agent tự động | `requests`, Flask, FastAPI, psycopg2 auto |
| **Service identity** | Labels trong metrics | `DD_SERVICE` env var | `OTEL_SERVICE_NAME` env var |
| **Multiple instances** | Prometheus labels | `DD_VERSION` | `service.instance.id` resource attribute |
| **Credentials** | Prometheus config file | `DD_API_KEY` env var | `APPLICATIONINSIGHTS_CONNECTION_STRING` env var |

**Cloud role name là critical cho Application Map.** Nếu 3 services cùng `service.name` → chỉ thấy 1 node trên map → mất hết topology visibility.

**Exam trap:** Precedence là `code > env var` cho connection string? **SAI** — env var `APPLICATIONINSIGHTS_CONNECTION_STRING` là recommended production approach. Precedence thực tế: code config override env var nếu explicitly set. Exam thường hỏi "production recommended" → env var.

---

## Checklist ghi nhớ cho AI-200

- [ ] `pip install azure-monitor-opentelemetry` → `configure_azure_monitor()` một lần startup
- [ ] Autoinstrumentation: requests, Flask/FastAPI, psycopg2, Azure SDK — không cần code
- [ ] `APPLICATIONINSIGHTS_CONNECTION_STRING` = recommended production config
- [ ] Code > env var precedence cho connection string
- [ ] Cloud role name = `service.namespace` + `service.name`
- [ ] Unique `service.name` per service → separate nodes trên Application Map
- [ ] `service.instance.id` = distinguish multiple instances của same service
- [ ] `OTEL_SERVICE_NAME` env var để set cloud role mà không đổi code

---

*Bài tiếp theo: Bài 3 — Configure spans and traces*
