# Bài 2 — Manage the Container App Lifecycle

> Khoá: AI-200 · ACA — Manage containers in Azure Container Apps

---

## Tổng quan

Lifecycle management là kỹ năng day-two quan trọng cho **cost control** và **incident response**. Với AI solution, bạn sẽ cần:

- **Pause service** khi upstream provider (model API) bị outage — tránh request fail liên tục
- **Restart** sau khi thay đổi config để nó có hiệu lực ngay
- **Stop** để giảm blast radius khi đang điều tra revision lỗi
- **Scale to zero** cho event-driven service hoặc workload không liên tục

---

## Ba loại action: Stop, Restart, Scale-to-zero

Đây là ba cơ chế khác nhau, dùng cho ba tình huống khác nhau:

| Action | Cơ chế | Dùng khi |
|---|---|---|
| **Scale to zero** | Scaling engine tự quyết định dựa trên config | Event-driven service, workload traffic thấp — tiết kiệm cost |
| **Stop** | Explicit action, đảm bảo không có replica nào chạy | Incident response — cần guarantee app không restart tự động |
| **Restart** | Force restart tất cả replica hiện tại | Config thay đổi cần hiệu lực ngay, process bị stuck/deadlock |

### Sự khác biệt quan trọng giữa Stop và Scale-to-zero

**Scale-to-zero** là behavior tự động của scaling engine — app vẫn configured để chạy, chỉ là hiện không có replica nào active. Khi có traffic mới, ACA tự động scale up.

**Stop** là explicit operational action — app bị told "đừng chạy replica nào". App sẽ không tự động scale up kể cả khi có traffic. Đây là sự khác biệt cốt lõi: stop đảm bảo không có replica nào khởi động trong khi bạn đang điều tra.

### Lưu ý đặc thù với AI service

AI service thường **load large model lúc startup** — có thể tốn hàng chục giây. Restart quá thường xuyên làm tăng cold start frequency và ảnh hưởng user experience. Dùng restart **có chủ đích**, không dùng như "magic fix" khi không hiểu nguyên nhân.

---

## Start và Stop — App-level action

Stop và start là action ở **cấp app** — ảnh hưởng toàn bộ container app, không phải chỉ một revision.

### Stop app

```bash
az containerapp stop \
    --name <app-name> \
    --resource-group <resource-group>
```

Sau khi stop, toàn bộ replica bị terminate và **không tự động restart**. App vẫn tồn tại trong Azure (config, secret, revision history còn nguyên) — chỉ không chạy.

### Start app

```bash
az containerapp start \
    --name <app-name> \
    --resource-group <resource-group>
```

Start lại app sau khi stop. ACA sẽ scale up replica theo config hiện tại.

### Khi nào dùng stop/start vs revision deactivate?

> **Stop là quá broad nếu vấn đề chỉ ở một revision.**

Nếu revision mới lỗi nhưng revision cũ vẫn healthy → dùng **revision deactivate** (bài trước) để ngừng revision lỗi trong khi revision cũ vẫn phục vụ user.

Stop toàn bộ app chỉ phù hợp khi:
- Upstream dependency bị outage hoàn toàn (model provider down, database unreachable)
- Phát hiện security incident cần isolate service ngay lập tức
- Maintenance window cần guarantee không có traffic

---

## Restart — Recovery từ bad runtime state

```bash
az containerapp restart \
    --name <app-name> \
    --resource-group <resource-group>
```

Restart force tất cả replica hiện tại restart — clear transient state, reload config.

### Khi nào restart hữu ích

- **Config change cần hiệu lực ngay:** Một số config thay đổi cần restart để áp dụng
- **Process stuck/deadlock:** App đang chạy nhưng không respond — có thể do deadlock hoặc memory leak transient
- **Transient dependency issue:** Kết nối đến database hoặc cache bị stale và không tự recover

### Restart KHÔNG phải substitute cho root cause analysis

> Pair restart với log inspection — xác định restart có thực sự fix vấn đề không, hay chỉ delay failure tiếp theo.

```bash
# Restart rồi xem log ngay để verify
az containerapp restart --name ai-api --resource-group rg-aca-demo
az containerapp logs show --name ai-api --resource-group rg-aca-demo --follow --tail 50
```

Nếu app fail lại sau vài phút, restart chỉ là bandaid — cần tìm nguyên nhân thật.

---

## Diagnose failing revision — Checklist

Khi revision lỗi, có một checklist các pattern phổ biến cần validate theo thứ tự:

### Bước 1: Confirm revision nào đang lỗi và health state

```bash
az containerapp revision list \
    --name ai-api \
    --resource-group rg-aca-demo \
    --query "[].{name:name,active:properties.active,health:properties.healthState}" \
    -o table
```

Output dạng table dễ đọc, so sánh nhanh revision nào Healthy, Unhealthy.

### Bước 2: Kiểm tra từng failure pattern

**1. Image pull failures**

Triệu chứng: Revision không bao giờ start, log có lỗi về image pull.

Nguyên nhân phổ biến:
- Registry credentials missing hoặc invalid (managed identity chưa được gán AcrPull)
- Image reference sai (typo trong digest, tag không tồn tại)
- Image name có chữ hoa (phải lowercase)

**2. Port mismatches**

Triệu chứng: Container start nhưng health probe fail, request trả 502/503.

Nguyên nhân: Container lắng nghe port X nhưng ACA ingress config expect port Y.

Kiểm tra: `az containerapp show --query properties.configuration.ingress.targetPort` so với port trong Dockerfile/startup command.

**3. Missing configuration**

Triệu chứng: App crash ngay khi start, log có lỗi về undefined variable hoặc missing config.

Nguyên nhân: Env var hoặc secret bị thiếu hoặc sai tên.

Kiểm tra: Compare env vars giữa revision hoạt động và revision lỗi bằng `revision show`.

**4. Probe failures**

Triệu chứng: App start và serve request được nhưng ACA coi là unhealthy, liên tục restart.

Nguyên nhân:
- Liveness/readiness probe trỏ sai path
- Probe timeout quá ngắn — AI service load model lâu, probe timeout trước khi model load xong
- Initial delay quá ngắn cho model loading

**5. Resource pressure**

Triệu chứng: Container start nhưng slow hoặc bị kill. Log có OOMKilled hoặc CPU throttling.

Nguyên nhân: Memory limit quá thấp cho model, hoặc CPU quá thấp gây throttling.

Kiểm tra: `az containerapp revision show` để xem resource allocation của revision.

---

## Tóm tắt quyết định — Dùng action nào?

```
Vấn đề ở revision cụ thể?
    ├── Có → revision deactivate (bài 1)
    └── Không (vấn đề toàn app) →
            ├── Process stuck/transient? → restart
            ├── Upstream outage, cần guarantee không chạy? → stop
            └── Traffic thấp, muốn tiết kiệm cost? → cấu hình scale-to-zero

Sau khi fix → start lại nếu đã stop
```

---

## Checklist ghi nhớ cho AI-200

- [ ] **Scale-to-zero** = auto behavior của scaling engine, app có thể tự scale up khi có traffic
- [ ] **Stop** = explicit action, đảm bảo không có replica nào chạy, không tự scale up
- [ ] **Restart** = force reload tất cả replica, dùng khi config thay đổi hoặc process stuck
- [ ] Stop là **app-level action** — ảnh hưởng toàn bộ app, không chỉ một revision
- [ ] Nếu vấn đề chỉ ở một revision → dùng **revision deactivate** thay vì stop toàn app
- [ ] Restart với AI service cần cẩn thận — model load lâu, restart thường xuyên tăng cold start
- [ ] Luôn **pair restart với log inspection** — verify fix hay chỉ delay failure
- [ ] `az containerapp stop/start/restart --name ... --resource-group ...`
- [ ] Checklist failure: image pull → port mismatch → missing config → probe failure → resource pressure
- [ ] Query nhanh health state: `revision list --query "[].{name:name,active:...,health:...}" -o table`

---

*Bài tiếp theo: Bài 3 — Monitor logs and troubleshoot issues*
