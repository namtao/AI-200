# Tổng hợp kiến thức — 02-ACA (3 Modules)

> AI-200 · Module 1: Deploy containers to ACA · Module 2: Manage containers in ACA · Module 3: Scale containers in ACA

---

## Bức tranh tổng thể

```
ACA vs App Service — khi nào dùng cái nào?

App Service:  1 container, 1 web app, đơn giản, không cần microservice
ACA:          nhiều service giao tiếp nội bộ, scale độc lập, không cần quản lý Kubernetes

ACA là Kubernetes được managed hoàn toàn — bạn không thấy node, pod, hay cluster.
```

**3 module = 3 giai đoạn vận hành:**

```
Module 1: Deploy          → Tạo Environment → Deploy app → Config runtime → Verify
Module 2: Day-two ops     → Update image → Revisions → Lifecycle → Monitor → Probes → Resources
Module 3: Scaling         → Scale rules → Event-driven (KEDA) → Custom scalers → Compute → Revision modes
```

---

## MODULE 1 — Deploy Containers to ACA

### Environment — Khái niệm trung tâm

Environment là **logical boundary** ảnh hưởng 3 thứ:
1. **Networking:** Apps trong cùng environment chia sẻ virtual network → giao tiếp nội bộ qua DNS
2. **Observability:** Tất cả app gửi log về cùng Log Analytics workspace
3. **Governance:** Policy áp dụng ở cấp environment

> Apps ở **khác environment** không thể giao tiếp nội bộ — dù cùng subscription.

```bash
az containerapp env create --name aca-env-demo --resource-group rg --location centralus
```

**External ingress** = public, nhận traffic từ internet  
**Internal ingress** = chỉ nhận traffic từ app trong cùng environment

### Deploy approaches

| Command | Dùng khi |
|---|---|
| `az containerapp up` | Prototype, first deploy — tự tạo resource |
| `az containerapp create` | Production, explicit, cần environment có sẵn |
| `az containerapp create --yaml` | Config phức tạp, cần code review, production |
| `az containerapp update` | Cập nhật → tạo **revision mới** |

Khi dùng `--yaml`, tất cả CLI flag khác **bị bỏ qua hoàn toàn**.  
Image repository name phải **lowercase** — chữ hoa gây pull failure.

```bash
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
```

### Runtime Settings — Env vars và Secrets

```
Non-sensitive → Environment variables   → --env-vars KEY=VALUE
Sensitive     → Secrets                 → az containerapp secret set --secrets key="value"
Bridge        → Secret reference        → --set-env-vars MY_VAR=secretref:<secret-name>
```

- `--env-vars` dùng với `create` · `--set-env-vars` dùng với `update` (không xoá biến khác)
- YAML cú pháp: `secretRef: <name>` (camelCase, không có `:`)
- YAML chỉ lưu **tên secret**, không bao giờ lưu giá trị
- Rotate secret: chỉ cần `secret set` lại, không rebuild/redeploy

### Registry Auth

```bash
# Managed identity (ACR + production)
az containerapp registry set -n app -g rg --server reg.azurecr.io --identity system

# Username/password (non-ACR hoặc dev)
az containerapp registry set -n app -g rg --server reg.io --username u --password p
```

Managed identity cần role **`AcrPull`** trên ACR. `--identity system` = system-assigned.

### Verify Deployments — 4 tool chain

```
1. az containerapp show          → config, ingress, FQDN
2. az containerapp logs show     → console log (app) / system log (platform)
3. az containerapp revision list → revision active, healthState, trafficWeight
4. az containerapp replica list  → instance đang chạy, runningState
```

- Console log: debug app error · System log: `--type system`, debug platform issue
- Real-time: `--follow --tail 30`
- Revision fields: `active`, `trafficWeight`, `healthState`, `provisioningState`
- `revision list --all` → bao gồm inactive revision

---

## MODULE 2 — Manage Containers in ACA

### Revision Model — Core concept

Mỗi lần `az containerapp update` → tạo **revision mới** (immutable snapshot).  
Revision cũ vẫn tồn tại cho đến khi bị deactivate hoặc xoá.

| State | Nghĩa |
|---|---|
| Active + traffic weight | Đang phục vụ request |
| Active + no traffic weight | Sẵn sàng nhưng không được route |
| Inactive | Không nhận traffic, vẫn tồn tại (evidence, rollback) |

**Thứ tự xử lý revision lỗi:**
```
Phát hiện lỗi → deactivate → investigate → fix → redeploy → delete
(Không skip, không delete trước investigate)
```

```bash
az containerapp update --image reg.io/app@sha256:<digest>   # tạo revision mới
az containerapp revision list -o table                       # overview nhanh
az containerapp revision show --revision <name>              # compare config
az containerapp revision deactivate --revision <name>        # bước đầu incident
az containerapp revision activate --revision <name>          # rollback
```

ACA auto-purge inactive revision khi vượt **100**, điều chỉnh bằng `--max-inactive-revisions`.

**Tags vs Digests:**
- Tag = mutable, dev/staging
- Digest (`@sha256:...`) = immutable, production bắt buộc

### Lifecycle Actions

| Action | Semantics | Dùng khi |
|---|---|---|
| `scale-to-zero` | Auto, sẽ tự scale up khi có traffic | Event-driven workload, tiết kiệm cost |
| `containerapp stop` | Explicit, **không tự restart** | Upstream outage, security incident |
| `containerapp restart` | Force reload tất cả replica | Config change, process stuck |

**Revision deactivate** ưu tiên hơn `stop` khi vấn đề chỉ ở một revision.

**Failure checklist:**
1. Image pull failure (registry auth, typo, chữ hoa)
2. Port mismatch (targetPort vs container port)
3. Missing env var/secret
4. Probe failure (sai path/port, initialDelay quá ngắn)
5. Resource pressure (OOM → tăng memory, throttle → tăng CPU)

### Monitoring — Log là cửa sổ duy nhất

ACA không có shell access vào container (khác App Service có Kudu). Log là tín hiệu diagnostic duy nhất.

**4 fields bắt buộc trong AI service log:**
- Correlation ID (trace qua nhiều service)
- Revision / Build ID (biết version nào đang chạy)
- Model version (quan trọng khi A/B test model)
- Latency breakdown (xác định bottleneck)

Không log raw document/prompt — log metadata và size.

```kusto
// Log Analytics KQL
ContainerAppConsoleLogs_CL
| where TimeGenerated > ago(1h)
| where Log_s contains "ERROR"
| project TimeGenerated, RevisionName_s, Log_s
```

### Health Probes — Readiness vs Liveness

| | Readiness | Liveness |
|---|---|---|
| Câu hỏi | Sẵn sàng nhận traffic? | Còn alive và không stuck? |
| Khi fail | Ngừng route traffic | **Restart replica** |
| Check external dep? | Nên có (DB, model loaded) | **Không** (DB down ≠ restart app) |
| AI use case | Guard model loading (30-120s) | Safety valve deadlock/memory |

```yaml
probes:
- type: Readiness
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 20
  periodSeconds: 10
  failureThreshold: 3

- type: Liveness
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 60   # Phải > model load time
  failureThreshold: 3
```

**4 nguyên nhân probe fail phổ biến:** sai port, sai path, initialDelay quá ngắn, dependency chậm.

### Resources & Scaling Basics

- **Total cost = per-replica cost × số replica**
- CPU thiếu → **throttle** (slow) · Memory thiếu → **OOM kill** (crash)
- Update resource → tạo revision mới

```bash
az containerapp update --cpu 1.0 --memory 2Gi
```

- `minReplicas: 1` → production AI API (tránh cold start)
- `minReplicas: 0` → background worker (tiết kiệm cost, chấp nhận cold start)

---

## MODULE 3 — Scale Containers in ACA

### Scale Rule Types

| Rule type | Trigger | Scale-to-zero | Dùng khi |
|---|---|---|---|
| **HTTP** | Concurrent requests/15s | ✅ | Synchronous API |
| **TCP** | Concurrent connections | ✅ | WebSocket, gRPC |
| **CPU** | CPU utilization % | ❌ min 1 replica | Compute-intensive |
| **Memory** | Memory utilization % | ❌ min 1 replica | Memory-intensive |
| **Event-driven** | Queue depth, stream lag | ✅ | Async background worker |

Default HTTP threshold: **10 concurrent requests/replica**  
Scale-up: ngay lập tức · Scale-down: chờ **300 giây**  
Khi nhiều rule: scale khi **bất kỳ rule nào** trigger, dùng **highest replica count**

### Event-Driven Scaling (KEDA)

ACA dùng KEDA, polling mỗi **30 giây**.

| Scaler | Scale theo | Lưu ý |
|---|---|---|
| `azure-servicebus` | Message count trong queue/topic | Formula: messages ÷ threshold |
| `azure-queue` | Queue length | Đơn giản hơn Service Bus |
| `azure-eventhub` | Partition lag | Max replicas = số partition |
| `kafka` | Consumer group lag | Max replicas = số partition |
| `redis` (Lists) | LLEN | Simple queue |
| `redis` (Streams) | Pending entries | Track in-progress work |
| `cron` | Time window | Predictive pre-scale |
| `prometheus` | PromQL query result | Custom business metrics |

```bash
# Service Bus scaler với managed identity
az containerapp create -n processor -g rg --environment env \
  --image reg.io/processor:v1 \
  --min-replicas 0 --max-replicas 30 \
  --user-assigned <IDENTITY_ID> \
  --scale-rule-name sb-scaling --scale-rule-type azure-servicebus \
  --scale-rule-metadata "queueName=orders" "namespace=sb-ns" "messageCount=5" \
  --scale-rule-identity <IDENTITY_ID>
```

**Auth:** Managed identity (`--scale-rule-identity`) recommended production. Secret-based cho dev.  
Managed identity cần RBAC role phù hợp (vd. **Azure Service Bus Data Receiver**).

### Cron + HTTP kết hợp

```yaml
scale:
  minReplicas: 0   # Scale-to-zero overnight
  maxReplicas: 20
  rules:
    - name: business-hours
      custom:
        type: cron
        metadata:
          timezone: "America/New_York"
          start: "0 8 * * 1-5"    # 8AM weekdays
          end: "0 18 * * 1-5"
          desiredReplicas: "5"    # Pre-scale 5 replicas
    - name: http-scaling
      http:
        metadata:
          concurrentRequests: "50"
```

### Compute Resources

**Memory ≥ CPU × 2 GiB** (bắt buộc):

| CPU | Min Memory |
|---|---|
| 0.25 | 0.5 GiB |
| 0.5 | 1.0 GiB |
| 1.0 | 2.0 GiB |
| 4.0 | 8.0 GiB (max Consumption) |

**Consumption vs Dedicated:**
- **Consumption:** Serverless, shared, max 4 CPU/8 GiB, pay per use
- **Dedicated:** Phải dùng khi: GPU, >4 CPU/8 GiB, consistent latency, compliance

### Revision Modes

| | Single (default) | Multiple |
|---|---|---|
| Active revision | 1 | Nhiều |
| Deploy mới | Auto deactivate cũ, shift 100% traffic | Cũ không bị deactivate, bạn control |
| Canary/traffic split | ❌ | ✅ |
| Operational overhead | Thấp | Cao hơn |

```bash
az containerapp update --revision-mode multiple
az containerapp ingress traffic set --revision-weight v1=80 v2=20
```

**Revision labels** = named URL trỏ thẳng đến revision cụ thể.  
Move label = atomic redirect, instant, không downtime.

**Revision-scope changes** (tạo revision mới): image, scale rules, env vars, template  
**Application-scope changes** (không tạo revision): secrets, ingress settings, traffic rules

**50/50 traffic split lâu dài** → mỗi revision thấy ít hơn ngưỡng scale-out → scaling rules không trigger đúng.

---

## CLI Commands Quick Reference

```bash
# === Setup ===
az extension add --name containerapp --upgrade
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights

# === Environment ===
az containerapp env create --name env --resource-group rg --location centralus
az containerapp env show --name env --resource-group rg

# === Deploy ===
az containerapp up --name app --resource-group rg --environment env \
    --image image:tag --target-port 80 --ingress external
az containerapp create --name app --resource-group rg --environment env \
    --image image:tag --ingress external --target-port 80
az containerapp update --name app --resource-group rg --image image@sha256:digest
az containerapp create --name app --resource-group rg --environment env --yaml file.yml

# === Config ===
az containerapp secret set -n app -g rg --secrets key="value"
az containerapp update -n app -g rg --set-env-vars KEY=VALUE SECRET=secretref:key
az containerapp registry set -n app -g rg --server reg.io --identity system

# === Verify ===
az containerapp show -n app -g rg --query properties.configuration.ingress.fqdn --output tsv
az containerapp logs show -n app -g rg --follow --tail 30
az containerapp logs show -n app -g rg --type system
az containerapp revision list -n app -g rg -o table
az containerapp revision list -n app -g rg --all
az containerapp revision show -n app -g rg --revision <name>
az containerapp replica list -n app -g rg

# === Lifecycle ===
az containerapp revision deactivate -n app -g rg --revision <name>
az containerapp revision activate -n app -g rg --revision <name>
az containerapp stop/start/restart -n app -g rg

# === Scaling ===
az containerapp update -n app -g rg --cpu 1.0 --memory 2Gi --min-replicas 1 --max-replicas 10
az containerapp update -n app -g rg --revision-mode multiple
az containerapp ingress traffic set -n app -g rg --revision-weight v1=80 v2=20

# === Scale rules ===
az containerapp create ... \
    --min-replicas 0 --max-replicas 10 \
    --scale-rule-name http-rule --scale-rule-type http \
    --scale-rule-http-concurrency 50
```

---

## Exam Traps — Những điểm hay nhầm

1. **Apps khác environment không giao tiếp nội bộ** — dù cùng subscription.

2. **Khi dùng `--yaml`, CLI flags khác bị bỏ qua hoàn toàn** — YAML là source of truth duy nhất.

3. **`--env-vars` vs `--set-env-vars`:** `env-vars` dùng khi create, `set-env-vars` khi update và không xoá biến khác.

4. **YAML chỉ lưu tên secret** (`secretRef: key-name`), không bao giờ lưu giá trị secret.

5. **CPU/Memory scale rules không thể scale về 0** — cần ít nhất 1 replica để đo. Chỉ HTTP và event-driven scale mới hỗ trợ scale-to-zero.

6. **Scale-to-zero ≠ Stop:** Scale-to-zero tự wake up khi có traffic. Stop thì không tự restart.

7. **Liveness probe không check external dependency** — DB down ≠ restart container. Liveness chỉ check "process còn alive không?"

8. **`initialDelaySeconds` của liveness phải > model load time** — nếu không → liveness fail khi model đang load → restart loop.

9. **Deactivate trước, investigate sau, delete cuối cùng** — không delete revision khi đang troubleshoot.

10. **Event Hubs và Kafka: max effective replicas = số partition** — replica thứ N+1 (N=partition count) không có partition để read.

11. **Revision-scope vs Application-scope:** Thay đổi image/env var/scale rules → revision mới. Thay đổi secrets/ingress/traffic rules → không tạo revision mới.

12. **50/50 traffic split lâu dài** → mỗi revision thấy 50% traffic → scale rule có thể không trigger đúng.

13. **Memory phải ≥ CPU × 2 GiB** — không phải tự do set.

14. **Image repository name phải lowercase** — chữ hoa gây pull failure trông như auth error.

---

## Assessment Summary

### Module 1 — Deploy
| Câu | Tình huống | Đáp án |
|---|---|---|
| 1 | Shared networking + log cho nhiều app | Container Apps environment |
| 2 | Config reviewable như code | `--yaml` với file trong source control |
| 3 | API key không lưu trong YAML | Secrets + `secretref:` |
| 4 | Confirm versioned change sau update | `az containerapp revision list` |
| 5 | First step khi container fail to start | `az containerapp logs show` |

### Module 2 — Manage
| Câu | Tình huống | Đáp án |
|---|---|---|
| 1 | Image traceability production | Digest (`@sha256:...`) |
| 2 | Ngừng traffic revision mà không xoá | `az containerapp revision deactivate` |
| 3 | Nguyên nhân phổ biến nhất readiness fail ngay | Sai port hoặc sai path trong probe config |
| 4 | First step khi nghi ngờ runtime exception | Stream log + filter by revision |
| 5 | Fix CPU throttling | Tăng per-replica CPU, reassess scaling |

### Module 3 — Scale
| Câu | Tình huống | Đáp án |
|---|---|---|
| 1 | Scale-to-zero cho queue processor | `--min-replicas 0` + azure-servicebus rule |
| 2 | Pre-scale 8AM + scale-to-zero overnight | Cron rule + HTTP rule kết hợp |
| 3 | 10% canary traffic | Multiple revision mode + traffic splitting |
| 4 | Auth cho KEDA scalers production | Managed identity (`--scale-rule-identity`) |

---

## ACA vs App Service — Khi nào dùng cái nào?

| | App Service | ACA |
|---|---|---|
| Số container service | 1 | Nhiều |
| Internal communication | ❌ | ✅ |
| Scale-to-zero | ❌ | ✅ |
| Canary deployment | 2 slots only | Nhiều revision + traffic % |
| Shell access (debug) | ✅ Kudu + SSH | ❌ chỉ có log |
| Event-driven scaling | ❌ | ✅ KEDA |
| GPU support | ❌ | ✅ Dedicated profile |
| Kubernetes underneath | ❌ | ✅ (managed, bạn không thấy) |
| Complexity | Thấp hơn | Cao hơn |

---

[← Assessment](./aca-m3-module-assessment.md) · [🏠 Mục lục](../README.md) · [Mục lục →](../README.md)
