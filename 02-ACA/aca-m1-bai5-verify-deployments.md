# Bài 5 — Verify Deployments with Logs and Status

> Khoá: AI-200 · ACA — Deploy containers to Azure Container Apps

---

## Tại sao verification quan trọng với AI service?

Deploy thành công ≠ app hoạt động đúng. Với AI service, startup failure có thể trông giống nhiều vấn đề khác nhau:

- Container không start → có thể là missing env var, hoặc model provider unreachable, hoặc registry auth fail
- App start nhưng trả lỗi → có thể là secret không được resolve, hoặc dependency endpoint sai
- App chạy nhưng chậm → có thể là cold start, scale-to-zero, hoặc replica không đủ

Verification giúp bạn xác nhận **deployment intent khớp với thực tế đang chạy**. ACA cung cấp 4 tool chính:

```
Verification tools
├── az containerapp show      → xem config và ingress (bước đầu tiên)
├── az containerapp logs show → xem container log (thường nhanh nhất)
├── az containerapp revision  → kiểm tra revision nào active
└── az containerapp replica   → kiểm tra instance đang chạy
```

> **Ghi nhớ:** ACA có hai loại log:
> - **Console logs** — output từ container của bạn (stdout/stderr) → dùng để debug app error
> - **System logs** — log từ platform ACA → dùng khi nghi ngờ vấn đề infrastructure

---

## Bước 1: Kiểm tra app config và ingress

Trước khi đào sâu vào log, verify config cơ bản — phát hiện vấn đề rõ ràng như ingress bị set sai.

```bash
az containerapp show -n ai-api -g rg-aca-demo
```

Output cho thấy toàn bộ config hiện tại của app: image đang dùng, ingress type (external/internal), FQDN, environment, resources...

**Dùng `--query` để lấy thông tin cụ thể:**

```bash
# Lấy FQDN để test endpoint
az containerapp show -n ai-api -g rg-aca-demo \
    --query properties.configuration.ingress.fqdn \
    --output tsv

# Lấy ingress type để kiểm tra external/internal
az containerapp show -n ai-api -g rg-aca-demo \
    --query properties.configuration.ingress.external
```

Sau khi có FQDN, gửi health check request để verify app respond:

```bash
curl https://<fqdn>/health
```

---

## Bước 2: Xem container logs

Container log thường là **cách nhanh nhất** để hiểu vấn đề. Với AI API, log thường hiện:
- HTTP binding error (port sai, bind vào localhost thay vì 0.0.0.0)
- Missing env var khi app khởi động
- Connection failure đến model provider, database
- Authentication error khi gọi external service

### Xem log gần đây (default)

```bash
az containerapp logs show -n ai-api -g rg-aca-demo
```

Trả về số dòng log mặc định (thường 20 dòng gần nhất). Đủ để xem startup log.

### Follow log real-time

```bash
az containerapp logs show -n ai-api -g rg-aca-demo \
    --follow --tail 30
```

- `--follow` — stream log liên tục như `tail -f`
- `--tail 30` — hiển thị 30 dòng cuối trước khi bắt đầu stream

Dùng khi cần quan sát behavior khi có traffic, test tải, hoặc debug vấn đề chỉ xảy ra lúc runtime.

### Xem system log (platform-level)

```bash
az containerapp logs show -n ai-api -g rg-aca-demo \
    --type system
```

Dùng khi nghi ngờ vấn đề ở tầng platform: image pull failure, container scheduling, networking... Không phải app code.

---

## Bước 3: Kiểm tra Revisions

Revision là versioned snapshot của config — mỗi lần update app tạo revision mới. Liệt kê revision giúp xác nhận:
- Update đã tạo revision mới chưa?
- Revision mới đã đạt trạng thái healthy chưa?
- Revision nào đang active (nhận traffic)?

### Liệt kê revision active

```bash
az containerapp revision list -n ai-api -g rg-aca-demo
```

### Liệt kê cả revision inactive (để debug)

```bash
az containerapp revision list -n ai-api -g rg-aca-demo --all
```

`--all` bao gồm cả revision cũ đã bị deactivate — hữu ích khi cần so sánh config giữa revision hoạt động và revision bị lỗi.

**Thông tin quan trọng trong output:**

| Field | Ý nghĩa |
|---|---|
| `name` | Tên revision (dùng để reference trong lệnh khác) |
| `active` | true/false — có đang nhận traffic không |
| `trafficWeight` | % traffic được route đến revision này |
| `healthState` | Healthy/Unhealthy/None |
| `provisioningState` | Provisioned/Deprovisioning/Failed |

---

## Bước 4: Kiểm tra Replicas

Replica là **instance đang chạy** của một revision. Với AI workload, replica behavior ảnh hưởng trực tiếp đến latency và throughput:

- **Scale-to-zero:** Không có replica → cold start khi có request
- **Scale-out:** Nhiều replica → request được distribute, giảm latency
- **Crash loop:** Replica liên tục restart → app không ổn định

### Liệt kê replica của revision active

```bash
az containerapp replica list -n ai-api -g rg-aca-demo
```

### Liệt kê replica của revision cụ thể

```bash
az containerapp replica list -n ai-api -g rg-aca-demo \
    --revision MyRevision
```

Dùng khi muốn kiểm tra replica của revision cũ (ví dụ: để debug tại sao revision đó fail).

**Thông tin quan trọng trong output:**

| Field | Ý nghĩa |
|---|---|
| `name` | Tên replica |
| `runningState` | Running/Stopped/Unknown |
| `containers` | Trạng thái từng container trong replica |

---

## Quy trình verification thực tế

Khi có sự cố với AI service, đi theo thứ tự này:

```
1. az containerapp show          → config đúng chưa? ingress đúng chưa? lấy FQDN
         ↓ nếu config ổn
2. az containerapp logs show     → app log có error gì không?
         ↓ nếu log không rõ ràng
3. az containerapp revision list → revision mới có healthy không?
         ↓ nếu revision healthy nhưng vẫn lỗi
4. az containerapp replica list  → replica đang chạy không? scale-to-zero? crash loop?
         ↓ nếu nghi ngờ platform issue
5. az containerapp logs show --type system  → platform log có gì không?
```

---

## So sánh với App Service diagnostic tools

| | ACA | App Service |
|---|---|---|
| Xem log | `az containerapp logs show` | `az webapp log tail` |
| Version/deployment | Revision (`az containerapp revision list`) | Deployment history |
| Instance | Replica (`az containerapp replica list`) | Instance (trong portal) |
| Config check | `az containerapp show` | `az webapp show` |
| Advanced console | Không có Kudu | Kudu (`<app>.scm.azurewebsites.net`) |
| Real-time log | `--follow` | `az webapp log tail` |

---

## Best Practices

### Bắt đầu bằng log

`az containerapp logs show` là lệnh đầu tiên khi có incident — startup error thường đủ rõ ràng trong log để xác định hướng debug.

### Dùng revision để validate rollout

Sau mỗi update, chạy `revision list` để confirm:
1. Revision mới đã được tạo chưa
2. `healthState` là Healthy
3. `trafficWeight` đã chuyển sang revision mới

### Inspect replica khi có incident

Khi app bị slow hoặc intermittent error:
- Replica count có đúng không?
- Có replica nào đang crash loop không?
- `runningState` của tất cả replica là Running không?

### Dùng `--query` để extract thông tin quan trọng

```bash
# Chỉ lấy revision name và health state
az containerapp revision list -n ai-api -g rg-aca-demo \
    --query "[].{name:name, health:properties.healthState, active:properties.active}"

# Chỉ lấy FQDN
az containerapp show -n ai-api -g rg-aca-demo \
    --query properties.configuration.ingress.fqdn \
    --output tsv
```

---

## Checklist ghi nhớ cho AI-200

- [ ] ACA có hai loại log: **console logs** (app output) và **system logs** (platform)
- [ ] Dùng console log để debug app error · Dùng system log khi nghi ngờ vấn đề platform
- [ ] Xem log: `az containerapp logs show` (mặc định console log)
- [ ] Real-time log: `--follow --tail <số-dòng>`
- [ ] System log: `--type system`
- [ ] `az containerapp show` để xem config + ingress + lấy FQDN
- [ ] `az containerapp revision list` → liệt kê revision active
- [ ] `az containerapp revision list --all` → bao gồm cả inactive revision
- [ ] Revision có field quan trọng: `active`, `trafficWeight`, `healthState`, `provisioningState`
- [ ] `az containerapp replica list` → liệt kê instance đang chạy
- [ ] `az containerapp replica list --revision <name>` → replica của revision cụ thể
- [ ] Scale-to-zero → không có replica → cold start khi có request
- [ ] Dùng `--query` để extract field cụ thể, phù hợp automation

---

*Bài tiếp theo: Exercise — Deploy a containerized backend API to Container Apps*
