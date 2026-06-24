# Bài 1 — Update Images and Manage Revisions Safely

> Khoá: AI-200 · ACA — Manage containers in Azure Container Apps

---

## Bối cảnh module 2

Module 1 tập trung vào **deploy lần đầu**. Module 2 là về **day-two operations** — những việc bạn làm hàng ngày sau khi app đã chạy production:

- Push image mới nhiều lần một ngày (tuning prompt, update model, fix bug)
- Revision mới đôi khi fail health probe — cần diagnose nhanh mà không gián đoạn user
- Giữ revision history gọn gàng, tránh tốn resource

Bài này tập trung vào: update image → tạo revision mới → verify health → quản lý traffic.

> **Lưu ý từ tài liệu:** Các lệnh trong module này là pattern — luôn validate flags với documentation hiện tại trước khi đưa vào automation.

---

## Tags vs Digests — Chọn cái nào cho production?

Đây là quyết định quan trọng với AI application vì ảnh hưởng đến **traceability** — khả năng truy vết chính xác artifact nào đang phục vụ request.

### Tags — Tiện nhưng có thể thay đổi

Tag như `dev`, `staging`, `v1.2` là mutable — ai đó có thể push image mới đè lên cùng tag. Hai lần deploy cùng tag có thể chạy code khác nhau.

Dùng tag khi: development, iteration nhanh, staging environment.

### Digests — Bất biến, audit-friendly

Digest `sha256:<hash>` là immutable — một khi image được push, digest đó gắn với image đó mãi mãi.

```
myregistry.azurecr.io/ai-api@sha256:a1b2c3d4e5f6...
```

Dùng digest khi: production deployment, cần audit trail, incident review.

**Tại sao digest quan trọng với AI:**

Khi có incident lúc 3 giờ sáng, bạn cần biết chính xác model version nào đang phục vụ request. Với digest, bạn có thể prove "lúc incident xảy ra, service đang chạy image này, với model weights này". Với tag, bạn chỉ biết tag tên là gì — không biết đó là build nào.

---

## Update image → Tạo revision mới

Khi update image, ACA tạo **revision mới** thay vì modify in-place. Revision mới có config mới, revision cũ vẫn tồn tại cho đến khi bạn xoá hoặc deactivate.

### Update image bằng digest (production)

```bash
az containerapp update \
    --name <app-name> \
    --resource-group <resource-group> \
    --image <registry>/<repo>@sha256:<digest>
```

Ví dụ thực tế với ACR:

```bash
az containerapp update \
    --name ai-api \
    --resource-group rg-aca-demo \
    --image myregistry.azurecr.io/ai-api@sha256:a1b2c3d4e5f6789...
```

### Verify image đã được record đúng

Sau khi update, confirm platform đã ghi nhận image đúng:

```bash
az containerapp show \
    --name ai-api \
    --resource-group rg-aca-demo \
    --query properties.template.containers
```

Output hiển thị image reference hiện tại trong container template — verify digest khớp với những gì bạn vừa deploy.

---

## Revision Modes — Single vs Multiple

Revision mode quyết định **bao nhiêu revision có thể active cùng lúc**.

### Single revision mode (mặc định)

Chỉ một revision active tại một thời điểm. Khi revision mới được tạo, revision cũ tự động bị deactivate.

**Phù hợp với:** Hầu hết usecase. Đơn giản, không cần quản lý traffic splitting.

**Hạn chế:** Không hỗ trợ canary deployment hoặc A/B testing.

### Multiple revision mode

Nhiều revision có thể active cùng lúc. Bạn kiểm soát bao nhiêu % traffic đến mỗi revision.

**Phù hợp với:**
- Canary deployment: 10% traffic đến revision mới, 90% ở revision cũ — validate trước khi shift hoàn toàn
- A/B testing: So sánh hai model version với traffic thật
- Gradual rollout: Tăng dần traffic đến revision mới

**Ví dụ với AI service:** Bạn vừa update model từ GPT-3.5 sang GPT-4. Thay vì shift 100% traffic ngay, bạn route 5% traffic đến revision mới để monitor latency và cost trước khi commit.

---

## "Active" có nghĩa là gì?

**Active** là operational state — revision đang **có thể nhận traffic**. Cần phân biệt:

| State | Ý nghĩa |
|---|---|
| Active + receiving traffic | Revision đang phục vụ request |
| Active + no traffic weight | Revision có thể nhận traffic nhưng không được route vào |
| Inactive | Revision không nhận traffic, nhưng **vẫn tồn tại** trong registry |
| Deleted | Revision đã bị xoá hoàn toàn |

**Inactive revision vẫn có giá trị:**
- Giữ lại để điều tra incident (log, config còn nguyên)
- Rollback plan: reactivate khi cần
- Evidence preservation: không mất artifact khi troubleshoot

---

## Liệt kê, điều tra, và quản lý Revisions

### Liệt kê revision (dạng table dễ đọc)

```bash
az containerapp revision list \
    --name ai-api \
    --resource-group rg-aca-demo \
    -o table
```

`-o table` format output thành bảng — dễ nhìn hơn JSON khi cần overview nhanh.

### Xem chi tiết một revision cụ thể

```bash
az containerapp revision show \
    --name ai-api \
    --resource-group rg-aca-demo \
    --revision <revision-name>
```

Dùng khi cần **so sánh config** giữa revision đang hoạt động và revision bị lỗi — tìm ra điểm khác biệt gây ra vấn đề.

### Deactivate revision — Bước đầu tiên khi có incident

```bash
az containerapp revision deactivate \
    --name ai-api \
    --resource-group rg-aca-demo \
    --revision <revision-name>
```

**Tại sao deactivate thay vì delete ngay?**

Deactivate là **safer first step** vì:
1. Revision ngay lập tức ngừng nhận traffic → giảm impact cho user
2. Config, log, và artifact vẫn còn → có thể điều tra nguyên nhân
3. Nếu cần rollback, reactivate lại ngay mà không cần rebuild
4. Delete là irreversible — chỉ làm khi chắc chắn không cần nữa

**Quy trình xử lý revision lỗi:**

```
Phát hiện revision mới fail
    → Deactivate revision mới (traffic quay về revision cũ)
    → Điều tra: xem log, so sánh config với revision cũ
    → Fix nguyên nhân
    → Deploy revision mới nữa
    → Sau khi stable → xoá revision lỗi cũ
```

### Xoá revision cũ (routine hygiene)

Sau khi incident được giải quyết và không cần revision cũ cho rollback:

```bash
az containerapp revision deactivate \
    --name ai-api \
    --resource-group rg-aca-demo \
    --revision <old-revision-name>
# Rồi mới xoá nếu cần
```

**Auto-purge:** ACA tự động purge inactive revision khi vượt quá **100 revision**. Bạn có thể điều chỉnh ngưỡng này bằng `--max-inactive-revisions` khi tạo/update app.

---

## Rollback — Quay về revision trước

Rollback trong ACA không phải "undo" — mà là **shift traffic về revision cũ** (vẫn còn tồn tại).

### Single revision mode rollback

Trong single mode, activate lại revision cũ sẽ deactivate revision hiện tại:

```bash
# Activate lại revision cũ
az containerapp revision activate \
    --name ai-api \
    --resource-group rg-aca-demo \
    --revision <old-revision-name>
```

### Tại sao ACA rollback nhanh hơn App Service?

Với App Service, rollback cần pull image lại. Với ACA revision model, revision cũ vẫn đang có sẵn infrastructure — activate lại gần như instant.

---

## Tóm tắt workflow update an toàn

```
1. Build image mới, tag bằng digest
   → myregistry.azurecr.io/ai-api@sha256:newdigest...

2. Update app → tạo revision mới
   → az containerapp update --image ...@sha256:newdigest...

3. Verify revision mới
   → az containerapp revision list -o table
   → Kiểm tra healthState, provisioningState

4. Nếu healthy → shift traffic (multiple mode) hoặc tự động (single mode)

5. Nếu unhealthy → deactivate revision mới
   → Traffic tự động quay về revision cũ
   → Điều tra log: az containerapp logs show

6. Sau khi stable → dọn dẹp revision cũ không cần nữa
```

---

## Best Practices

### Dùng digest cho production

```bash
# Production — immutable, auditable
--image myregistry.azurecr.io/ai-api@sha256:a1b2c3...

# Development — tiện nhưng mutable
--image myregistry.azurecr.io/ai-api:dev
```

### Verify revision health trước khi shift traffic

Đừng assume revision mới healthy chỉ vì update command thành công. Kiểm tra `healthState` trong revision list trước khi route production traffic.

### Deactivate trước, investigate sau, delete cuối cùng

Thứ tự này bảo tồn evidence và giữ rollback option luôn available.

### Định nghĩa retention strategy

Với team deploy nhiều lần một ngày, inactive revision tích lũy nhanh. Set `--max-inactive-revisions` phù hợp để tránh ACA auto-purge revision bạn vẫn cần.

---

## Bản chất bài này là gì?

**Một câu:** Revision model biến mỗi deployment thành một immutable snapshot có thể rollback, traffic split, và điều tra — thứ mà Docker và App Service không có.

### So sánh: Deploy mới trong Docker vs ACA

| | Docker | App Service | ACA Revision |
|---|---|---|---|
| Khi update image | Container cũ bị replace | Deployment history (portal only) | Revision mới tạo, revision cũ còn nguyên |
| Rollback | Pull image tag cũ + redeploy | Swap deployment slots | Activate lại revision cũ — instant |
| So sánh config | Không có | Khó | `revision show` cả hai revision rồi so sánh |
| Traffic split (canary) | Cần load balancer riêng | Deployment slots (2 slots) | Traffic weights trên nhiều revision |
| Evidence sau incident | Log còn, config mất | Hạn chế | Revision config, log, image reference đều còn |

**Điểm quan trọng nhất:** ACA rollback = activate revision cũ (infrastructure sẵn sàng → gần như instant). App Service rollback = pull image lại (có thể tốn vài phút).

---

## Checklist ghi nhớ cho AI-200

- [ ] **Tag** = mutable, tiện cho dev · **Digest** = immutable, dùng cho production AI
- [ ] Digest cho phép prove chính xác artifact nào phục vụ request trong incident
- [ ] `az containerapp update --image ..@sha256:<digest>` → tạo revision mới
- [ ] Verify image sau update: `az containerapp show --query properties.template.containers`
- [ ] **Single revision mode** → chỉ một revision active, đơn giản
- [ ] **Multiple revision mode** → nhiều revision active, hỗ trợ canary/traffic splitting
- [ ] **Active** = revision có thể nhận traffic · **Inactive** = không nhận traffic nhưng vẫn tồn tại
- [ ] Inactive revision vẫn có giá trị: điều tra, rollback, evidence
- [ ] `az containerapp revision list -o table` → overview nhanh tất cả revision
- [ ] `az containerapp revision show --revision <name>` → detail để so sánh config
- [ ] `az containerapp revision deactivate` → bước đầu khi revision lỗi, **trước** khi investigate
- [ ] Deactivate → investigate → fix → redeploy → delete (không skip bước)
- [ ] ACA auto-purge inactive revision khi vượt **100**, điều chỉnh bằng `--max-inactive-revisions`

---

*Bài tiếp theo: Bài 2 — Manage the container app lifecycle*
