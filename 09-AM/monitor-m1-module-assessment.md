# Module Assessment — Instrument an App with OpenTelemetry

> Khoá: AI-200 · Azure Monitor — Instrument an app with OpenTelemetry

---

## Câu 1

**What standard does OpenTelemetry use to propagate trace context across service boundaries?**

- OpenTracing B3 headers
- W3C TraceContext using `traceparent` headers ✅
- OpenCensus binary propagation

**Đáp án: W3C TraceContext using `traceparent` headers**

Khi service A gọi service B, service A include `traceparent` header:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             {version}-{trace-id-32hex}-{parent-span-id-16hex}-{flags}
```

Service B đọc header, tạo span mới với **cùng trace ID** và caller's span ID làm parent. Tất cả spans qua tất cả services kết nối thành 1 trace.

Tại sao các đáp án còn lại sai:
- **OpenTracing B3 headers:** B3 là format của Zipkin/OpenTracing (trước khi OpenTelemetry ra đời). OpenTelemetry dùng W3C TraceContext là standard mới hơn, được IETF standardize.
- **OpenCensus binary propagation:** OpenCensus là predecessor của OpenTelemetry (merge từ OpenTracing + OpenCensus). Binary propagation không phải HTTP header format. OpenTelemetry không dùng format này.

---

## Câu 2

**When you configure the Azure Monitor connection string in a production environment, which approach is recommended?**

- Embed it directly in source code using the code-based configuration option.
- Store it in a `config.py` settings file that is deployed alongside the application code.
- Set the `APPLICATIONINSIGHTS_CONNECTION_STRING` environment variable. ✅

**Đáp án: Environment variable**

```bash
export APPLICATIONINSIGHTS_CONNECTION_STRING="InstrumentationKey=00000000...;IngestionEndpoint=https://eastus-0.in.applicationinsights.azure.com/"
```

Distro tự động pick up mà không cần code changes. Lợi ích:
- Sensitive config không trong source code (không risk leak qua git)
- Dễ thay đổi per environment mà không rebuild
- Consistent với 12-factor app principles

Tại sao các đáp án còn lại sai:
- **Embed in source code:** Documentation explicitly state "least recommended for production because it embeds credentials in source code." Connection string in git repo = security risk.
- **Store in `config.py`:** Tốt hơn inline code nhưng config file vẫn được deploy alongside application → same risk nếu file accessible, và vẫn cần code change khi rotate. Environment variables là standard practice cho production secrets.

---

## Câu 3

**When using `tracer.start_as_current_span()` in Python and an unhandled exception propagates out of the `with` block, what does the OpenTelemetry SDK do automatically?**

- Records the exception on the span and sets the span status to error. ✅
- Suppresses the exception and marks the span as successful to avoid false alerts.
- Leaves the span status as unset and doesn't record any exception details.

**Đáp án: Records exception + sets status to error**

```python
with tracer.start_as_current_span("GenerateEmbedding") as span:
    # Nếu exception propagate ra khỏi block:
    # → SDK tự động: span.record_exception(ex) + span.set_status(StatusCode.ERROR)
    embedding = generate_embedding(input_text)
    # Span kết thúc khi block exits
```

Nếu muốn manual control (custom error description):
```python
try:
    embedding = generate_embedding(input_text)
except Exception as ex:
    span.record_exception(ex)
    span.set_status(Status(StatusCode.ERROR, "Embedding generation failed"))
    raise
```

Tại sao các đáp án còn lại sai:
- **Suppresses exception and marks successful:** Điều này sẽ làm cho observability vô nghĩa — nếu errors không được reported, không thể detect và debug failures. SDK không bao giờ suppress exceptions.
- **Leaves status unset, no exception details:** Đây là điều SDK làm nếu exception được **caught** bên trong block (không propagate out). Khi exception **propagates out** của `with` block, SDK automatically records it. Đây là distinction quan trọng.

---

## Câu 4

**A Python AI application creates a span using `tracer.start_as_current_span("CallLlmApi", kind=SpanKind.CLIENT)`. In which Application Insights table does this span appear?**

- `requests`
- `traces`
- `dependencies` ✅

**Đáp án: `dependencies`**

```python
with tracer.start_as_current_span("CallLlmApi", kind=SpanKind.CLIENT) as span:
    span.set_attribute("llm.model", "gpt-4o")
    response = call_llm(prompt)
```

`SpanKind.CLIENT` = outgoing call → **dependencies table** trong App Insights.

Full mapping:
| SpanKind | App Insights table |
|---|---|
| SERVER | requests |
| **CLIENT** | **dependencies** |
| INTERNAL | dependencies |
| CONSUMER | requests |
| PRODUCER | dependencies |

Tại sao các đáp án còn lại sai:
- **`requests`:** Server-side spans (SpanKind.SERVER — incoming HTTP requests) và Consumer spans (SpanKind.CONSUMER — queue message processing) go vào requests. CLIENT span là **outgoing** → dependencies.
- **`traces`:** `traces` table chứa **log records** từ Python logging module hoặc OTel Logging API. Không phải spans. Span types go vào requests hoặc dependencies depending on SpanKind.

---

## Câu 5

**Which field in Application Insights telemetry tables corresponds to the OpenTelemetry trace ID and enables correlation of all telemetry items in a distributed trace?**

- `cloud_RoleName`
- `operation_Id` ✅
- `id`

**Đáp án: `operation_Id`**

OpenTelemetry Trace ID ↔ Application Insights `operation_Id`. Tất cả tables (requests, dependencies, exceptions, traces) share `operation_Id` → đây là basis để join và correlate distributed traces:

```kql
exceptions
| join kind=inner (
    requests | project requestName = name, operation_Id, requestRoleName = cloud_RoleName
) on operation_Id
| project timestamp, exceptionType = type, requestName, cloud_RoleName = requestRoleName
```

Tại sao các đáp án còn lại sai:
- **`cloud_RoleName`:** Identifies **which service** generated telemetry (`service.name` + `service.namespace` từ OTel). Hữu ích để filter by service nhưng không link individual requests qua services.
- **`id`:** Là span-level identifier (tương ứng Span ID, không phải Trace ID). Unique per telemetry item nhưng không shared qua trace — không phải correlation key.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Context propagation standard | W3C TraceContext (`traceparent` header) |
| 2 | Connection string production config | `APPLICATIONINSIGHTS_CONNECTION_STRING` env var |
| 3 | Unhandled exception trong span block | SDK auto records exception + sets ERROR status |
| 4 | SpanKind.CLIENT → App Insights table | `dependencies` |
| 5 | OTel Trace ID ↔ App Insights field | `operation_Id` |

---

*Module hoàn thành.*

---

[← Bài 5](./monitor-m1-bai5-debug-traces.md) · [🏠 Mục lục](../README.md) · [Bài 1 →](./monitor-m2-bai1-kql-basics.md)
