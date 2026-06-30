# Bài 3 — Tag and Version Images

> Khoá: AI-200 · Module: Implement container application hosting on Azure

---

## Tại sao tagging strategy quan trọng?

Bạn đã biết tag là nhãn gắn vào image. Nhưng **cách bạn đặt tên tag** ảnh hưởng trực tiếp đến:

- **Deployment reliability** — liệu tất cả node trong cluster có chạy đúng cùng một image không?
- **Rollback capability** — khi có sự cố, bạn có thể quay lại phiên bản cũ không?
- **Traceability** — khi incident xảy ra lúc 3 giờ sáng, bạn có biết đang chạy image nào không?
- **Storage cost** — image cũ tích lũy theo thời gian tốn tiền như thế nào?

Không có một strategy duy nhất đúng cho mọi tình huống — bài này trình bày các lựa chọn và khi nào dùng cái nào.

---

## Stable Tags vs Unique Tags

Đây là sự phân biệt cơ bản nhất trong tagging strategy.

### Stable Tags — tag được tái sử dụng

Stable tag là tag **bị ghi đè mỗi khi push image mới cùng tên**. Ví dụ điển hình: `latest`, `v1`, `v1.2`, `stable`, `nightly`.

```
Lần push 1: inference-api:latest → trỏ vào image A
Lần push 2: inference-api:latest → trỏ vào image B  (image A vẫn còn nhưng mất tag)
Lần push 3: inference-api:latest → trỏ vào image C  (image B vẫn còn nhưng mất tag)
```

**Khi nào stable tag hữu ích:**

- **Base image nhận security patch:** Bạn muốn tất cả image phụ thuộc tự động nhận bản vá khi base image được cập nhật. Ví dụ: tag `python:3.11` của Docker Hub là stable tag — khi có patch mới, Docker Hub push lại vào tag đó, và ACR base image trigger sẽ tự rebuild image của bạn.

- **Development environment:** Developer muốn luôn có bản mới nhất mà không cần cập nhật config. Kéo `inference-api:dev` là luôn có code mới nhất.

- **Continuous delivery trong staging:** Môi trường staging muốn tự động nhận mọi build mới để kiểm thử liên tục.

**Vấn đề của stable tag trong production:**

Hãy hình dung Kubernetes cluster của bạn có 5 node đang chạy `inference-api:latest`. Node thứ 6 được thêm vào sau khi ai đó push image mới — node mới pull `latest` và nhận phiên bản khác với 5 node cũ. Bây giờ cluster của bạn đang chạy **hai phiên bản model khác nhau cùng lúc** mà không ai hay biết. Với AI inference, điều này có thể tạo ra kết quả không nhất quán rất khó debug.

---

### Unique Tags — tag không bao giờ bị tái sử dụng

Unique tag là tag **chỉ gắn với một image duy nhất, không bao giờ bị ghi đè**. Ví dụ: `v1.2.0-build456`, `20260102-abc123f`, `v1.2.0`.

```
Lần push 1: inference-api:v1.0.0    → image A (mãi mãi)
Lần push 2: inference-api:v1.0.1    → image B (mãi mãi)
Lần push 3: inference-api:v1.1.0    → image C (mãi mãi)
```

**Khi nào unique tag là lựa chọn đúng:**

- **Production deployment:** Tất cả node trong cluster đều pull đúng cùng image đó, không có ngoại lệ.

- **Audit trail:** Bạn cần chứng minh với security team hoặc khách hàng rằng vào ngày X, hệ thống đang chạy chính xác image nào.

- **Rollback:** Khi version mới có vấn đề, bạn chỉ cần đổi deployment manifest về tag cũ — image đó vẫn còn nguyên trong registry.

- **Compliance:** Trong các ngành yêu cầu audit (tài chính, y tế), bạn cần chứng minh được image nào đang chạy tại mọi thời điểm.

**Đánh đổi của unique tag:**

Mỗi lần release version mới, bạn phải cập nhật deployment configuration (Kubernetes manifest, Helm values,...) để trỏ sang tag mới. Đây thực ra được coi là **ưu điểm** trong production — mọi thay đổi phải là cố ý, không có update ngầm.

---

## Semantic Versioning — đặt tên version có ý nghĩa

Semantic versioning (SemVer) là quy ước đặt tên version theo định dạng `MAJOR.MINOR.PATCH`. Quy ước này được dùng rộng rãi trong phần mềm vì nó **truyền đạt bản chất của thay đổi** chỉ qua con số.

### Ý nghĩa của từng phần

**MAJOR** — Tăng khi có **breaking change**, tức là thay đổi làm hỏng người dùng hiện tại nếu không cập nhật theo.

Ví dụ với AI inference API:
- Đổi tên field trong JSON request/response
- Xoá một endpoint
- Thay đổi format input của model (ví dụ từ base64 sang URL)

**MINOR** — Tăng khi thêm **tính năng mới nhưng backward compatible**, tức là người dùng cũ vẫn dùng được bình thường.

Ví dụ:
- Thêm endpoint mới `/predict/v2` trong khi `/predict/v1` vẫn hoạt động
- Thêm optional parameter vào request
- Hỗ trợ thêm model mới

**PATCH** — Tăng khi **sửa bug hoặc security patch**, không thay đổi API.

Ví dụ:
- Sửa lỗi preprocessing tính sai kết quả
- Cập nhật dependency có lỗ hổng bảo mật
- Tối ưu performance không đổi output

### Ví dụ progression thực tế

```
inference-api:1.0.0    # Ra mắt lần đầu
inference-api:1.0.1    # Sửa bug trong bước tiền xử lý dữ liệu
inference-api:1.0.2    # Vá lỗ hổng bảo mật trong dependency
inference-api:1.1.0    # Thêm endpoint /predict/batch mới
inference-api:1.2.0    # Thêm hỗ trợ model GPT-4o-mini
inference-api:2.0.0    # Đổi format response (breaking change)
```

### Kết hợp stable tag và unique tag cho flexibility

Một pattern hay dùng là duy trì **cả hai loại tag cùng lúc** cho cùng một image, để người dùng tự chọn mức độ cập nhật:

```
inference-api:1          # Stable tag → luôn trỏ vào bản 1.x.x mới nhất
inference-api:1.1        # Stable tag → luôn trỏ vào bản 1.1.x mới nhất
inference-api:1.1.0      # Unique tag → mãi mãi là bản 1.1.0 này
```

- Team muốn nhận security patch tự động: dùng `inference-api:1.1`
- Team muốn kiểm soát tuyệt đối: dùng `inference-api:1.1.0`
- Team muốn nhận mọi feature mới trong major version: dùng `inference-api:1`

---

## Unique Tag Patterns — các cách tạo tag duy nhất

Trong thực tế, unique tag thường được tạo tự động bởi CI/CD pipeline. Có 4 pattern phổ biến:

### Pattern 1: Build ID

```
inference-api:build-4567
```

Build ID là số hiệu do hệ thống CI/CD tự sinh (GitHub Actions run number, Azure DevOps build ID,...).

**Ưu điểm:** Từ tag này, bạn biết chính xác pipeline run nào tạo ra image, có thể vào xem log build, test results, và các artifact liên quan.

**Dùng khi:** Team dùng CI/CD system tập trung và muốn link image → pipeline run.

### Pattern 2: Git Commit SHA

```
inference-api:abc123f
```

Git commit hash (thường dùng 7 ký tự đầu của short hash).

**Ưu điểm:** Từ tag này, bất kỳ ai có quyền truy cập repo đều có thể `git show abc123f` để xem chính xác code tạo ra image đó. Đây là link trực tiếp nhất giữa image và source code.

**Dùng khi:** Muốn traceability mạnh nhất giữa image và source code, đặc biệt hữu ích khi debug issue liên quan đến logic code.

### Pattern 3: Timestamp

```
inference-api:20260102-143022
```

Ngày và giờ build (thường theo UTC).

**Ưu điểm:** Nhìn vào danh sách image, bạn thấy ngay thứ tự thời gian và có thể nhanh chóng xác định image nào mới nhất mà không cần tra cứu hệ thống nào.

**Dùng khi:** Muốn dễ dàng xác định tuổi của image chỉ bằng mắt.

### Pattern 4: Combined — kết hợp nhiều thông tin

```
inference-api:v1.2.0-build4567-abc123f
```

Kết hợp semantic version + build ID + git hash trong một tag.

**Ưu điểm:** Trong một incident lúc 3 giờ sáng, bạn có ngay: version semantic (`v1.2.0`), số pipeline run (`build4567`), và commit hash (`abc123f`) — đủ để tra cứu mọi thứ mà không cần biết thêm thông tin nào khác.

**Dùng khi:** Production system đòi hỏi traceability cao, đặc biệt trong các ngành regulated.

### Dùng với ACR Tasks

Khi build bằng ACR Tasks, biến `{{.Run.ID}}` tự động tạo identifier duy nhất:

```bash
az acr build --registry myregistry \
  --image inference-api:v1.2.0-{{.Run.ID}} .
```

Kết quả: `inference-api:v1.2.0-cb1`, `inference-api:v1.2.0-cb2`,... — mỗi build có tag riêng biệt.

---

## Vấn đề với tag `latest`

Tag `latest` có behavior đặc biệt trong Docker: khi bạn `docker pull` hoặc `docker push` mà không chỉ định tag, Docker **tự động dùng `latest`**.

Sự tiện lợi này gây ra nhiều vấn đề nghiêm trọng trong production:

**Vấn đề 1 — Inconsistent deployment**

Node A pull `latest` lúc 9:00 → nhận image version X.
Node B được thêm vào lúc 14:00 → pull `latest` → nhận image version Y (ai đó đã push build mới).
Cluster giờ đang chạy 2 version khác nhau.

**Vấn đề 2 — Unintentional update**

Dev push build mới lên `latest` để test. Kubernetes tự động pull khi restart pod → production đột nhiên chạy code chưa được review kỹ.

**Vấn đề 3 — Khó troubleshoot**

Khi có bug, bạn không biết chính xác image nào đang chạy vì `latest` có thể là bất kỳ build nào.

### Cách đúng trong production

Luôn chỉ định explicit tag trong Kubernetes manifest:

```yaml
# Tránh dùng trong production
image: myregistry.azurecr.io/inference-api:latest

# Luôn dùng explicit version
image: myregistry.azurecr.io/inference-api:v1.2.0
```

Nếu workflow của bạn bắt buộc phải dùng `latest`, hãy giới hạn nó trong **development environment** và đổi sang explicit tag trước khi deploy lên staging/production.

---

## Lock Image — khoá image tránh xoá nhầm

ACR cho phép bạn **khoá một image** để ngăn việc xoá hoặc ghi đè, kể cả khi có quyền admin.

### Tại sao cần lock?

Hình dung tình huống: bạn đang chạy `inference-api:v1.2.0` trong production với SLA 99.9%. Một script cleanup tự động chạy và xoá image đó vì nó "trông có vẻ cũ". Kubernetes cluster bây giờ không thể restart pod vì image không còn. Đây là incident thực tế xảy ra ở nhiều tổ chức.

Lock image ngăn chặn điều này.

### Lock image

```bash
az acr repository update \
  --name myregistry \
  --image inference-api:v1.2.0 \
  --write-enabled false
```

Sau khi lock, image này có các đặc tính:

- **Không thể xoá** — kể cả admin cũng không xoá được trực tiếp
- **Không thể ghi đè** — push image mới với tag `v1.2.0` sẽ bị từ chối
- **Không bị retention policy xoá** — script cleanup tự động bỏ qua image locked
- **Deployment an toàn** — production workload chắc chắn image luôn available

### Unlock khi muốn retire

Khi version đó không còn dùng nữa và muốn dọn dẹp:

```bash
az acr repository update \
  --name myregistry \
  --image inference-api:v1.2.0 \
  --write-enabled true
```

Sau khi unlock, image có thể bị xoá bình thường.

**Best practice:** Lock image ngay sau khi deploy lên production. Unlock chỉ khi đã xác nhận toàn bộ workload đã chuyển sang version mới và không còn rollback plan nào cần version cũ nữa.

---

## Dọn dẹp Untagged Images

### Untagged image là gì và tại sao tốn tiền?

Khi bạn push image mới với một stable tag đã có, tag đó chuyển sang trỏ image mới. Image cũ vẫn còn trong registry nhưng không có tag nào — gọi là **untagged manifest** hay "orphan image".

```
Push v1: inference-api:dev → image A có tag
Push v2: inference-api:dev → image B có tag, image A mất tag → trở thành untagged
Push v3: inference-api:dev → image C có tag, image B mất tag → trở thành untagged
...
```

Sau vài tháng CI/CD chạy liên tục, bạn có thể tích lũy hàng trăm untagged image. Chúng không có ích gì (không có tag để pull) nhưng vẫn chiếm storage và tính tiền.

### Cách 1: Dọn thủ công với `acr purge`

`acr purge` là một command chạy như container trong ACR Tasks:

```bash
az acr run --registry myregistry \
  --cmd "acr purge --filter 'inference-api:.*' --untagged --ago 30d" \
  /dev/null
```

Giải thích từng phần:

- `acr purge` — tool cleanup tích hợp sẵn trong ACR
- `--filter 'inference-api:.*'` — regex chỉ định repository cần dọn. `inference-api:.*` nghĩa là repository tên `inference-api`, mọi tag. Dùng `.*:.*` để dọn toàn bộ registry.
- `--untagged` — chỉ xoá image không có tag (không đụng đến image có tag)
- `--ago 30d` — chỉ xoá image cũ hơn 30 ngày (bảo vệ image mới tạo gần đây)
- `/dev/null` — không cần source file (tương tự `az acr run` ở bài 2)

### Cách 2: Scheduled task — tự động dọn theo lịch

Tạo ACR Task chạy cleanup mỗi tuần:

```bash
az acr task create \
  --registry myregistry \
  --name cleanup-untagged \
  --cmd "acr purge --filter '.*:.*' --untagged --ago 7d" \
  --schedule "0 0 * * 0" \
  --context /dev/null
```

Giải thích:

- `--filter '.*:.*'` — dọn toàn bộ registry (mọi repository, mọi tag pattern)
- `--ago 7d` — xoá untagged image cũ hơn 7 ngày
- `--schedule "0 0 * * 0"` — cron expression: chạy lúc 00:00 UTC mỗi Chủ Nhật (`0` cuối = day of week 0 = Sunday)
- `--context /dev/null` — không cần source file

> **Tại sao dùng `--ago 7d` chứ không phải `--ago 0d`?**
> Giữ lại 7 ngày gần nhất là lưới an toàn: nếu có incident và bạn cần rollback về build 3 ngày trước (chưa có tag nào), image đó vẫn còn. Nếu xoá ngay thì mất cơ hội này.

### Cách 3: Retention Policy (Premium tier only)

Với Premium tier, bạn có thể cấu hình **retention policy ở cấp registry** — đơn giản hơn scheduled task vì chỉ cần set một lần, áp dụng tự động cho toàn bộ registry.

Thay vì tạo và quản lý ACR Task riêng, bạn chỉ cần vào Azure Portal → registry → Policies → Retention và đặt số ngày muốn giữ untagged manifest.

> **Ghi nhớ cho thi:** Retention policy chỉ có ở **Premium tier**. Basic và Standard phải dùng `acr purge` + scheduled task.

---

## So sánh 3 cách quản lý lifecycle

| | Manual purge | Scheduled task | Retention policy |
|---|---|---|---|
| Cách chạy | Thủ công, một lần | Tự động theo lịch | Tự động, luôn bật |
| Cần setup | Không | Có (tạo task) | Không (chỉ set số ngày) |
| Tier yêu cầu | Basic+ | Basic+ | **Premium only** |
| Áp dụng cho | Repository cụ thể (filter) | Repository theo filter | Toàn bộ registry |
| Độ linh hoạt | Cao (tùy chỉnh filter) | Cao | Thấp (chỉ set số ngày) |

---

## Best Practices tổng hợp

### Dùng unique tag cho production

```yaml
# Kubernetes manifest — production
image: myregistry.azurecr.io/inference-api:v1.2.0-build4567-abc123f
```

Unique tag đảm bảo tất cả node trong cluster chạy đúng cùng một image. Không có surprise update.

### Dành stable tag cho base image

```dockerfile
# Dockerfile của application image
FROM python:3.11   # stable tag → tự động nhận patch khi python:3.11 được update
```

Base image trigger của ACR sẽ tự động rebuild application image khi `python:3.11` có bản vá mới.

### Lock image đang chạy production

```bash
# Ngay sau khi deploy thành công lên production
az acr repository update \
  --name myregistry \
  --image inference-api:v1.2.0 \
  --write-enabled false
```

### Implement retention policy ngay từ đầu

Đừng chờ đến khi storage bill tăng cao mới nghĩ đến cleanup. Tạo scheduled cleanup task ngay khi setup registry:

```bash
az acr task create \
  --registry myregistry \
  --name weekly-cleanup \
  --cmd "acr purge --filter '.*:.*' --untagged --ago 7d" \
  --schedule "0 0 * * 0" \
  --context /dev/null
```

### Bao gồm build metadata trong tag

Tối thiểu nên có semantic version + git hash:

```
inference-api:v1.2.0-abc123f
```

Khi có incident, hai thông tin này đủ để: biết breaking change hay không (`v1.2.0`), và tìm đúng code trong repo (`abc123f`).

### Document tagging scheme cho cả team

Ghi rõ trong README hoặc wiki:
- Tag nào là stable, tag nào là unique
- Format tag production là gì (`v{semver}-{gitsha}`)
- Khi nào được phép dùng `latest` (chỉ local dev, không bao giờ production)
- Quy trình lock/unlock image

---

## ACR vs Docker — Điểm khác biệt về tagging

Về **khái niệm** tag, ACR không thay đổi gì so với Docker — cơ chế stable/unique tag, vấn đề của `latest`, semantic versioning đều hoạt động y hệt. Điểm khác biệt thực sự là những tính năng **lifecycle management** mà ACR bổ sung xung quanh:

| Feature | ACR | Docker Hub / local Docker |
|---|---|---|
| **Lock image** (`--write-enabled false`) | Có — ngăn xoá/ghi đè kể cả admin | Không có |
| **`acr purge`** | Tool cleanup tích hợp sẵn | Phải tự viết script |
| **Scheduled cleanup task** | Có (`--schedule` cron trên ACR Tasks) | Không có built-in |
| **Retention policy** | Có (Premium tier) | Không có |
| **`{{.Run.ID}}`** | Biến tự động tạo unique identifier mỗi build | Không có |

**Nếu bạn đã biết Docker tagging:** Phần lý thuyết đầu bài (stable tag, unique tag, SemVer, vấn đề `latest`) là ôn lại — không có gì mới. Phần thực sự mới là **lock image** và **lifecycle management** (purge, scheduled task, retention policy).

---

## Checklist ghi nhớ cho AI-200

- [ ] **Stable tag** bị ghi đè khi push image mới cùng tag → dùng cho base image và dev environment
- [ ] **Unique tag** không bao giờ bị tái sử dụng → bắt buộc dùng cho production deployment
- [ ] Semantic versioning: **MAJOR** (breaking) · **MINOR** (tính năng mới, backward compatible) · **PATCH** (bug fix, security)
- [ ] 4 unique tag patterns: **Build ID**, **Git commit SHA**, **Timestamp**, **Combined**
- [ ] `{{.Run.ID}}` trong ACR Tasks tự động tạo unique identifier cho mỗi build
- [ ] Tag `latest` gây inconsistent deployment vì các node pull vào thời điểm khác nhau có thể nhận image khác nhau
- [ ] Luôn dùng **explicit tag** trong Kubernetes manifest production, không dùng `latest`
- [ ] Lock image: `--write-enabled false` → không thể xoá, không thể ghi đè, không bị retention policy xoá
- [ ] Unlock image: `--write-enabled true` → dùng khi muốn retire version đó
- [ ] `acr purge --untagged --ago Nd` → xoá untagged image cũ hơn N ngày
- [ ] Scheduled cleanup task: dùng cron expression, chạy `acr purge` định kỳ
- [ ] **Retention policy chỉ có ở Premium tier** — Basic/Standard phải dùng scheduled purge task

---

[← Bài 2](./acr-m1-bai2-acr-tasks.md) · [🏠 Mục lục](../README.md) · [Assessment →](./acr-m1-module-assessment.md)
