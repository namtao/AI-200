# Bài 1 — Explore OpenTelemetry and Its Role in Observability

> Khoá: AI-200 · Azure Monitor — Instrument an app with OpenTelemetry

---

## Observability là gì?

Khả năng understand internal state của system qua external outputs. Distributed AI apps (RAG pipeline, 4 microservices) cần observability vì issues có thể xuất phát từ bất kỳ service nào.

---

## 3 Pillars of Observability

| Pillar | Provides | Tells you |
|---|---|---|
| **Distributed tracing** | Full request path qua services | **Where** delays/errors occur |
| **Metrics** | Aggregate numbers (counts, rates, percentiles) | **When** something changed |
| **Logs** | Timestamped event records | **Why** something happened |

---

## OpenTelemetry

Vendor-neutral, open-source observability framework (CNCF). Instrument once → export đến bất kỳ backend nào (Azure Monitor, Jaeger, Prometheus, Grafana...) không đổi instrumentation code.

**Components:**
- **APIs** — Interfaces tạo + manage telemetry (stable, embed in code)
- **SDKs** — Implement APIs, handle batching/sampling/resource detection
- **Instrumentation libraries** — Auto-capture telemetry từ common frameworks
- **Exporters** — Serialize + transmit đến backends

**Azure Monitor OpenTelemetry Distro** — Bundle SDK + Azure Monitor exporters + common libraries → single package.

---

## Traces, Spans, Context Propagation

**Trace** = Complete record của request's journey. Gồm nhiều spans.

**Span** = Named, timed operation. Contains:
- **Trace ID** — Shared by ALL spans in same trace (links everything)
- **Span ID** — Unique per span
- **Parent span ID** — Defines hierarchy
- Name, timestamps, attributes (key-value), status

**Context propagation** — W3C TraceContext standard. `traceparent` HTTP header:
```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             {version}-{trace-id-32hex}-{parent-span-id-16hex}-{flags}
```

---

## OpenTelemetry ↔ Application Insights Mapping

| OTel concept | Python | App Insights term |
|---|---|---|
| Span (Server) | `SpanKind.SERVER` | **Request** (requests table) |
| Span (Client) | `SpanKind.CLIENT` | **Dependency** (dependencies table) |
| Span (Internal) | `SpanKind.INTERNAL` | Dependency |
| Span (Consumer) | `SpanKind.CONSUMER` | Request |
| Span (Producer) | `SpanKind.PRODUCER` | Dependency |
| **Trace ID** | `span.get_span_context().trace_id` | **Operation ID** |
| Span Attributes | `span.set_attribute()` | **customDimensions** |

---

## Bản chất bài này là gì?

**Một câu:** OpenTelemetry là lingua franca của observability — instrument một lần, vendor không quan trọng, và mọi span đều được xâu lại thành trace qua `traceparent` header.

### So sánh với Jaeger / Zipkin / Datadog APM

| | Jaeger / Zipkin | Datadog APM | Azure Monitor / OTel |
|---|---|---|---|
| **Instrumentation** | Vendor-specific SDK | Datadog agent + SDK | OTel SDK (CNCF standard) |
| **Trace format** | OpenTracing B3 headers | Datadog proprietary | **W3C TraceContext** (`traceparent`) |
| **Backend lock-in** | Locked to Jaeger/Zipkin | Locked to Datadog | Instrument once → swap backend |
| **AI app suitability** | Tốt cho microservices | Tốt, có ML features | Native Azure integration |
| **Context propagation** | B3 multi-header | x-datadog-trace-id | **traceparent** single header |

**OTel không phải backend — nó là cái lớp giữa app và backend.** Jaeger/Zipkin/Datadog đều có thể nhận OTel data qua OTLP exporter. Azure Monitor Distro chỉ là OTel bundle được pre-packaged với Azure exporter.

**Exam trap:** Trace ID trong OTel = `operation_Id` trong App Insights, KHÔNG phải `id` (id = span ID). Nếu exam hỏi "correlation field" → `operation_Id`.

---

## Checklist ghi nhớ cho AI-200

- [ ] 3 pillars: Traces (where) · Metrics (when) · Logs (why)
- [ ] OpenTelemetry = CNCF vendor-neutral, instrument once → any backend
- [ ] Azure Monitor OpenTelemetry Distro = bundle SDK + exporter + libraries
- [ ] Trace = full request path · Span = individual operation
- [ ] Trace ID = shared by all spans in same trace
- [ ] `traceparent` header = W3C TraceContext, propagates trace context
- [ ] Server/Consumer spans → **requests table** · Client/Internal/Producer → **dependencies table**
- [ ] `span.set_attribute()` → **customDimensions** column in App Insights
- [ ] Trace ID ↔ **Operation ID** in App Insights

---

[← Mục lục](../README.md) · [🏠 Mục lục](../README.md) · [Bài 2 →](./monitor-m1-bai2-setup.md)
