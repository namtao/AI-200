# Bài 5 — Choose and Apply Revision Modes

> Khoá: AI-200 · ACA — Scale containers in Azure Container Apps

---

## Revision và Scaling — Mối liên hệ

Revision là **immutable snapshot** của container app config. Hai loại change:

| Change type | Tạo revision mới? | Ví dụ |
|---|---|---|
| **Revision-scope** | ✅ Có | Image, scale rules, env vars, template section |
| **Application-scope** | ❌ Không | Secrets, ingress settings, traffic splitting rules |

Mỗi revision **scale độc lập** theo scale rules của nó. Khi traffic split 50/50 giữa 2 revision → mỗi revision scale theo 50% traffic của nó → total resource có thể cao hơn 1 revision xử lý 100% traffic.

---

## Single Revision Mode (Default)

Chỉ **một revision active** tại một thời điểm.

**Zero-downtime deployment tự động:**
```
Deploy revision mới
    → Platform giữ revision cũ chạy và nhận traffic
    → Revision mới start, pass health checks
    → Traffic shift sang revision mới
    → Revision cũ deactivate tự động
```

Không cần config thêm — platform tự quản lý transition.

**Phù hợp:** Hầu hết deployments thông thường, không cần traffic splitting.

**Hạn chế:** Không support canary deployment — 100% traffic đến revision hiện tại, không thể test với subset.

---

## Multiple Revision Mode

Nhiều revision **active cùng lúc**, bạn kiểm soát traffic distribution.

```bash
az containerapp update \
  --name order-api \
  --resource-group rg-ecommerce \
  --revision-mode multiple
```

**Trong multiple mode:**
- Deploy mới → tạo revision mới, **revision cũ không tự deactivate**
- Bạn quyết định khi nào shift traffic và khi nào deactivate revision cũ
- **Active revision:** có thể nhận traffic, scale theo rules
- **Inactive revision:** không nhận traffic, không consume resource, vẫn tồn tại
- Tự động purge inactive revision khi vượt **100** (oldest first)

**Phù hợp:** Canary deployment, A/B testing, gradual rollout, blue-green deployment.

**Trade-off:** Cần operational attention — phải manage traffic distribution và cleanup revision cũ.

---

## Traffic Splitting

Distribute traffic theo **percentage weights**, tổng = 100%.

```bash
az containerapp ingress traffic set \
  --name order-api \
  --resource-group rg-ecommerce \
  --revision-weight order-api--v1=80 order-api--v2=20
```

80% traffic → v1, 20% → v2. Distribution là probabilistic (random per request).

### Canary deployment workflow

```
Deploy v2 → 0% traffic ban đầu
    ↓
Set 5-10% traffic đến v2 → monitor errors và latency
    ↓
Tăng dần: 10% → 25% → 50% → 100%
    ↓
Deactivate v1 sau khi confident với v2
```

### Traffic to latest revision

```bash
# Set weight cho "latest" thay vì revision name cụ thể
az containerapp ingress traffic set \
  --name order-api \
  --resource-group rg-ecommerce \
  --revision-weight latest=20 order-api--v1=80
```

Deployment mới tự động nhận 20% traffic mà không cần update traffic rules.

---

## Revision Labels — Direct Access URL

Label tạo **named endpoint** trỏ thẳng đến revision cụ thể, bất kể traffic splitting config.

**Use cases:**
- Test revision mới trực tiếp trước khi đưa vào traffic splitting
- Blue-green: label `blue` = production, `green` = staging → switch production bằng cách move label
- Share URL với QA team mà không ảnh hưởng production traffic

**Label naming rules:**
- Bắt đầu bằng chữ cái
- Chỉ lowercase alphanumeric và dashes
- Không có consecutive dashes
- ≤ 64 ký tự

**Move label** là atomic operation — redirect ngay lập tức, không downtime.

Label URL và main app URL hoạt động độc lập:
- Main URL → theo traffic splitting rules
- Label URL → luôn đến revision được labeled

---

## Scaling với Multiple Revision — Vấn đề cần lưu ý

**Scenario:** 2 revision, `minReplicas: 1`, split 50/50.

```
Revision v1: minReplicas=1, scale rule trigger tại 100 req/s
Revision v2: minReplicas=1, scale rule trigger tại 100 req/s

Total traffic: 200 req/s → mỗi revision nhận 100 req/s
→ Không revision nào vượt 100 req/s threshold → không scale!
→ Nhưng nếu single revision: 200 req/s > 100 → scale up
```

Split traffic 50/50 kéo dài = hiệu ứng scaling không như mong đợi. **Transition nhanh qua các mức %** thay vì dừng lại ở 50/50 lâu.

---

## Quy trình Blue-Green Deployment

```
Initial state:
  label "blue" → v1 (production, 100% traffic)
  label "green" → v2 (staging, 0% traffic)

Deploy v2 → test qua label URL "green"
    ↓
Confident → move label "blue" sang v2
    ↓
"blue" URL giờ trỏ v2 → instant switch, không downtime
    ↓
Deactivate v1
```

---

## Checklist ghi nhớ cho AI-200

- [ ] **Revision-scope changes** tạo revision mới (image, scale rules, env vars)
- [ ] **Application-scope changes** không tạo revision mới (secrets, ingress, traffic rules)
- [ ] Mỗi revision scale **độc lập** theo traffic share của nó
- [ ] **Single revision mode** (default): một active, zero-downtime deployment tự động
- [ ] **Multiple revision mode**: nhiều active, cần manual traffic management
- [ ] Enable multiple mode: `az containerapp update --revision-mode multiple`
- [ ] Traffic splitting: `az containerapp ingress traffic set --revision-weight v1=80 v2=20`
- [ ] Canary: bắt đầu 5-10%, tăng dần, deactivate cũ sau khi confident
- [ ] **Revision labels** = named URL trỏ thẳng đến revision, bất kể traffic splitting
- [ ] Label move là **atomic** — redirect ngay lập tức
- [ ] 50/50 split kéo dài → mỗi revision thấy ít traffic hơn → scale rules có thể không trigger đúng
- [ ] Inactive revision limit: **100**, oldest bị auto-purge

---

*Module Assessment tiếp theo*
