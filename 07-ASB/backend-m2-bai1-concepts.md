# Bài 1 — Understand Azure Event Grid Concepts and Event-Driven Patterns

> Khoá: AI-200 · Event Grid — Develop event-driven AI workflows with Azure Event Grid

---

## Event Grid là gì?

Fully managed **event-routing service** — kết nối event sources với event handlers ở sub-second latency, per-event pricing. Không cần polling — events được push đến endpoint khi state change xảy ra.

**Lợi ích so với polling:** Giảm end-to-end latency từ minutes xuống seconds. Loại bỏ wasted compute trên empty checks. Decoupling: add consumer mới mà không sửa producer.

---

## 2 Delivery Models

| Model | Behavior |
|---|---|
| **Push** | Event Grid gửi HTTP POST đến endpoint (webhook, Function, Service Bus) |
| **Pull** | Consumer connect và đọc events at own pace |

---

## 5 Core Components

### Events
Lightweight notification mô tả state change ("blob created", "model trained"). Không chứa full data — subscriber retrieve resource riêng nếu cần. Giữ payload nhỏ để deliver nhanh.

### Event Sources
Azure services (Blob Storage, Key Vault, Container Registry, Event Hubs) hoặc custom apps. AI app có thể vừa là consumer (subscribe storage events) vừa là source (publish events khi training xong).

### Topics — 3 loại

| Type | Ai tạo | Ai publish | Dùng khi |
|---|---|---|---|
| **System topics** | Event Grid tự tạo khi subscribe Azure resource | Chỉ Azure service | React to infrastructure events (blob created) |
| **Custom topics** | User tạo explicitly | Your application | Application-level events (inference completed) |
| **Namespace topics** | Part of Event Grid namespace | Your application | Push + pull, MQTT support |

### Event Subscriptions
Config resource xác định: events nào receive, filter nào, destination endpoint nào. Nhiều subscriptions trên cùng topic = fan-out. Filter theo event type, subject path, data attributes.

### Event Handlers
Destinations nhận events:
- **Azure Functions** — serverless, auto-scaling
- **Azure Event Hubs** — high-throughput streaming
- **Azure Service Bus** — enterprise messaging, guaranteed ordering
- **Webhooks** — any HTTP endpoint
- **Azure Storage queues** — async worker processing

---

## Event-Driven Patterns cho AI

**Reactive data processing:** `Microsoft.Storage.BlobCreated` → trigger AI pipeline khi new training data/documents arrive. Không cần polling loop.

**Pipeline stage coordination:** Embeddings done → publish event → trigger indexing step. Scale mỗi stage độc lập. Tự nhiên observability.

**Model lifecycle management:** Training complete → validation → deployment promotion. Downstream services subscribe mà không cần coupling.

---

## System Topics vs Custom Topics

| | System Topics | Custom Topics |
|---|---|---|
| Source | Azure services (Blob, Key Vault...) | Your application code |
| Creation | Auto-created | Explicitly by you |
| Schema | Predefined by Azure | You define |
| Use case | Infrastructure events | App-level events (inference, pipeline) |

**Example:** System topic → `BlobCreated` khi user upload → classification service process → publish custom event `ContentClassified` → notification service + analytics subscribe independently.

---

## Bản chất bài này là gì?

**Một câu:** Event Grid là event router push-based — kết nối event sources với handlers ở sub-second latency, per-event pricing, loại bỏ polling — khác với Service Bus (message broker) và Event Hubs (streaming).

### So sánh với Service Bus / Event Hubs / AWS EventBridge

| | Azure Event Grid | Azure Service Bus | Azure Event Hubs | AWS EventBridge |
|---|---|---|---|---|
| Mục đích | Event routing (notification) | Message queueing (task) | High-throughput streaming | Event bus (routing) |
| Payload size | Max 1 MB | 256 KB / 100 MB | 1 MB | 256 KB |
| Delivery | Push (HTTP POST) hoặc Pull | Pull (Peek-lock) | Pull (consumer group) | Push (targets) |
| Ordering | Không đảm bảo | Sessions = FIFO | Per-partition ordering | Không |
| Retention | 24h retry window | Configurable (days) | 1-90 ngày | 24h |
| Pricing | Per event | Per operation | Per throughput unit | Per event |
| Fan-out | Event subscription | Topic + subscriptions | Consumer groups | Rules + targets |
| Filter | Event type, subject, data | SQL filter, correlation | Không | Content-based rules |

**Khi nào dùng Event Grid vs Service Bus:** Event Grid cho "something happened, notify" (lightweight notifications, state changes). Service Bus cho "do this task reliably" (AI inference jobs, guaranteed processing).

**Exam trap:** Application code không thể publish đến system topics — chỉ Azure services mới publish lên system topics. Để emit application events, phải tạo custom topic.

---

## Checklist ghi nhớ cho AI-200

- [ ] Event Grid = push-subscribe event routing, per-event pricing, sub-second latency
- [ ] **Push** delivery = Event Grid sends HTTP POST · **Pull** = consumer reads at own pace
- [ ] 5 components: Events, Sources, Topics, Subscriptions, Handlers
- [ ] System topics: auto-created, Azure service publishes · Custom topics: user-created, app publishes
- [ ] Cannot publish directly to system topics
- [ ] Event handler options: Functions, Event Hubs, Service Bus, Webhooks, Storage queues
- [ ] Handlers phải idempotent — Event Grid = **at-least-once** delivery

---

*Bài tiếp theo: Bài 2 — Work with event schemas and properties*
