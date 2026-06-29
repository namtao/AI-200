# Bài 1 — Understand Azure Functions Hosting and Scaling for AI Workloads

> Khoá: AI-200 · Azure Functions — Build serverless AI backends with Azure Functions

---

## Cold Start

Delay khi platform allocate instance mới, load runtime, initialize dependencies trước khi handle first request. **Per-instance event** — chỉ ảnh hưởng request đầu tiên đến instance mới, không phải mọi request.

**AI workloads amplify cold start:** Large ML libraries (`azure-ai-documentintelligence`, `azure-cosmos`), model config loading, connection warm-up — có thể thêm nhiều giây ngoài base platform delay.

---

## Hosting Plans

### Flex Consumption — Recommended default

- **Linux-only**
- Per-function scaling (mỗi trigger type scale độc lập)
- Instance memory: **512 MB, 2,048 MB (default), 4,096 MB**
- **Always-ready instances** — giữ N warm instances cho specific functions
- Scale up đến 1,000 instances
- Charge: active execution time + always-ready baseline

### Premium

- **Eliminate cold starts hoàn toàn** — ít nhất 1 pre-warmed worker always running
- Custom Linux image support
- Larger compute sizes
- Dùng khi: continuous/near-continuous traffic, custom containers, larger compute needed

### Các plans khác

| Plan | Khi nào |
|---|---|
| Dedicated (App Service) | Underutilized App Service capacity đã có |
| Container Apps | Need GPU, custom OS packages |
| **Consumption (Linux)** | **Retire sau Sept 2028 — dùng Flex Consumption thay thế** |

---

## Per-function Scaling (Flex Consumption)

- **HTTP triggers** scale together (cùng instances)
- **Queue, Service Bus, Timer** mỗi loại scale **độc lập** trên instance set riêng
- → Queue surge không compete với HTTP handling

---

## Instance Memory và Concurrency

| Memory | vCPU | Dùng khi |
|---|---|---|
| 512 MB | Proportional | Lightweight, high-volume event processors |
| **2,048 MB (default)** | **1 vCPU** | **Hầu hết scenarios** |
| 4,096 MB | **2 vCPU** | Large models, memory-intensive data, higher CPU |

**Concurrency:** Thấp = nhiều instances, nhiều resources per invocation. Cao = ít instances, shared resources. AI inference: **lower concurrency** thường cho predictable performance hơn.

---

## HTTP Timeout — 230 Giây

HTTP Load Balancer timeout = **230 giây** — bất kể hosting plan hay `functionTimeout` setting. AI inference tasks > 230s → caller timeout.

**Giải pháp: Async request-reply pattern**

```python
@app.route(route="process-document", methods=["POST"], auth_level=func.AuthLevel.FUNCTION)
@app.service_bus_queue_output(arg_name="queue_msg", queue_name="document-jobs", connection="ServiceBusConnection")
def accept_document(req: func.HttpRequest, queue_msg: func.Out[str]) -> func.HttpResponse:
    job_id = str(uuid.uuid4())
    document_url = req.get_json().get("document_url")

    queue_msg.set(json.dumps({"job_id": job_id, "document_url": document_url}))

    return func.HttpResponse(
        json.dumps({"job_id": job_id, "status_url": f"/api/job-status/{job_id}"}),
        status_code=202,          # 202 Accepted — processing async
        mimetype="application/json"
    )
```

HTTP function → validate + enqueue → return **202** với status URL. Service Bus trigger → long-running processing (không có HTTP timeout).

---

## Bản chất bài này là gì?

**Một câu:** Azure Functions hosting plans = trade-off giữa cost và cold start — Flex Consumption là default mới (per-function scaling, always-ready), Premium eliminate cold start hoàn toàn, HTTP timeout cứng 230s cần async pattern cho long AI tasks.

### So sánh với AWS Lambda / Google Cloud Functions / Container Apps

| | Azure Functions Flex Consumption | Azure Functions Premium | AWS Lambda | Google Cloud Functions |
|---|---|---|---|---|
| Cold start | Có (always-ready giảm thiểu) | Không (pre-warmed) | Có | Có |
| Scale to zero | Có | Không (min 1 instance) | Có | Có |
| Per-function scaling | Có (Flex) | Không (per-app) | Có | Có |
| Max memory | 4,096 MB | Lớn hơn | 10,240 MB | 16,384 MB |
| HTTP timeout | **230s (Azure LB, không đổi được)** | 230s (same) | 15 phút | 60 phút |
| Custom container | Không | Có | Có | Có |
| Always-ready config | Per function | Per app (all instances) | Provisioned concurrency | Min instances |
| GPU support | Không | Không | Không | Không (Container Apps) |

**Tại sao Flex Consumption > Consumption (legacy):** Per-function scaling (Queue surge không làm chậm HTTP), always-ready per-function (tiết kiệm hơn Premium), Linux-only nhưng không giới hạn với hầu hết AI workloads.

**Exam trap:** HTTP timeout = **230 giây** là Azure Load Balancer limit — không thể override bằng `functionTimeout` setting trong host.json. Đây là hard ceiling. AI inference > 230s PHẢI dùng async pattern (return 202, enqueue).

---

## Checklist ghi nhớ cho AI-200

- [ ] Cold start = per-instance, không phải per-request
- [ ] AI workloads có cold start cao hơn do large dependencies
- [ ] **Flex Consumption** = recommended default, Linux-only, per-function scaling
- [ ] **Premium** = no cold start, always 1+ pre-warmed worker
- [ ] **Consumption (Linux) retire Sept 2028** → dùng Flex Consumption
- [ ] Always-ready instances: pay để giữ N warm instances (Flex Consumption)
- [ ] 4,096 MB = 2 vCPU, 2,048 MB = 1 vCPU (CPU proportional với memory)
- [ ] HTTP timeout = **230 giây** (Azure Load Balancer, không thể override)
- [ ] Long AI tasks: return **202 Accepted** + enqueue → async processing

---

*Bài tiếp theo: Bài 2 — Set up the local development environment*
