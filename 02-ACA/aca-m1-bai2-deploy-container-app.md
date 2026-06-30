# Bài 2 — Deploy a Container App Using the Azure CLI and YAML

> Khoá: AI-200 · ACA — Deploy containers to Azure Container Apps

---

## Tổng quan

Bài này bao gồm **hai cách deploy** container app lên ACA:

```
Deploy approaches
├── az containerapp up     → nhanh, tự tạo resource, dùng cho prototype/first deploy
├── az containerapp create → explicit, kiểm soát rõ ràng hơn
├── az containerapp update → cập nhật app → tạo revision mới
└── YAML-based deploy      → config lưu trong file, reviewable, dùng cho production
```

---

## Chuẩn bị Azure CLI

Trước khi deploy, cần đảm bảo CLI đã sẵn sàng. Bỏ qua bước này dễ gây lỗi khó hiểu, đặc biệt khi service thêm hoặc thay đổi command options.

```bash
# 1. Đăng nhập Azure
az login

# 2. Upgrade CLI lên version mới nhất
az upgrade

# 3. Cài hoặc upgrade ACA extension
az extension add --name containerapp --upgrade

# 4. Đăng ký resource providers (chỉ cần làm một lần trên subscription)
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
```

**Tại sao cần đăng ký resource providers?**

Azure cần biết subscription của bạn muốn dùng dịch vụ nào. `Microsoft.App` là namespace cho Azure Container Apps, `Microsoft.OperationalInsights` là cho Log Analytics (dùng để collect logs). Nếu chưa đăng ký, lệnh tạo environment sẽ fail.

---

## Cách 1: `az containerapp up` — Deploy nhanh nhất

`az containerapp up` là lệnh **all-in-one** — nó tự tạo environment, resource group (nếu chưa có), và deploy app trong một bước. Phù hợp khi:
- Prototype, thử nghiệm nhanh
- Lần deploy đầu tiên khi chưa biết cần cấu hình gì
- Demo, workshop

```bash
az containerapp up \
    --name my-container-app \
    --resource-group rg-aca-demo \
    --location centralus \
    --environment aca-env-demo \
    --image mcr.microsoft.com/k8se/quickstart:latest \
    --target-port 80 \
    --ingress external \
    --query properties.configuration.ingress.fqdn
```

Giải thích từng tham số:
- `--name` — tên container app
- `--resource-group` — resource group (tự tạo nếu chưa có)
- `--location` — Azure region
- `--environment` — tên environment (tự tạo nếu chưa có)
- `--image` — container image sẽ deploy
- `--target-port 80` — port mà container lắng nghe bên trong
- `--ingress external` — cho phép traffic từ internet
- `--query properties.configuration.ingress.fqdn` — in ra URL của app ngay sau khi deploy

**`--query`** dùng JMESPath syntax để lọc output JSON — ở đây lấy FQDN (Fully Qualified Domain Name) để biết URL truy cập app.

> **Lưu ý quan trọng về image name:** Repository name trong image path phải là **lowercase**. Ví dụ `mcr.microsoft.com/k8se/quickstart` phải viết thường toàn bộ. Nếu có chữ hoa, sẽ gây image pull failure — lỗi này trông giống authentication error, dễ nhầm.

---

## Cách 2: `az containerapp create` — Deploy explicit

Khi cần kiểm soát rõ ràng hơn, tạo environment trước (như bài 1) rồi dùng `create`:

```bash
az containerapp create \
    --name ai-api \
    --resource-group rg-aca-demo \
    --environment aca-env-demo \
    --image mcr.microsoft.com/k8se/quickstart:latest \
    --ingress external \
    --target-port 80
```

**Khi nào dùng `create` thay vì `up`?**
- Cần align app vào environment có sẵn (của team platform)
- Muốn kiểm soát từng bước trong CI/CD pipeline
- Cần đảm bảo consistent defaults giữa các environment (dev/staging/prod)
- Sẽ thêm nhiều tham số hơn (env vars, secrets, registry auth — các bài tiếp theo)

---

## Cập nhật App — Revision model

Khi update app, ACA **tạo một revision mới** thay vì modify in-place. Đây là khái niệm quan trọng của ACA.

### Revision là gì?

Revision là một **immutable snapshot** của configuration app tại một thời điểm. Mỗi khi bạn thay đổi image hoặc config, ACA tạo revision mới — revision cũ vẫn còn.

Điều này cho phép:
- **Validate revision mới** trước khi chuyển toàn bộ traffic
- **Rollback** về revision cũ nếu có vấn đề
- **Traffic splitting** — ví dụ 10% traffic đến revision mới, 90% ở revision cũ (canary deployment)

Với AI service, đây đặc biệt quan trọng vì model update có thể thay đổi behavior của API.

### Update image

```bash
az containerapp update \
    --name ai-api \
    --resource-group rg-aca-demo \
    --image myregistry.azurecr.io/ai-api:v2
```

Lệnh này tạo revision mới với image `v2`. Revision cũ (với `v1`) vẫn tồn tại cho đến khi bạn xoá.

Trong thực tế, bạn thường kết hợp update image với update config cùng lúc — thêm env vars mới, thay đổi resource limits, v.v.

---

## Cách 3: YAML-based deployment — Cho production

### Tại sao dùng YAML?

Khi AI service có nhiều setting — feature flags, dependency endpoints, scaling config, secrets references — quản lý bằng CLI flags trở nên khó bảo trì. YAML giải quyết điều này:

- **Reviewable:** Config thay đổi có thể review qua Pull Request như code
- **Repeatable:** Deploy cùng YAML file ra nhiều environment cho kết quả nhất quán
- **Version controlled:** Lịch sử thay đổi config được lưu trong Git
- **Single source of truth:** Khi dùng `--yaml`, tất cả flags CLI khác bị bỏ qua — YAML là nguồn duy nhất

### Deploy với YAML

```bash
# Tạo app mới từ YAML
az containerapp create \
    --name ai-api \
    --resource-group rg-aca-demo \
    --environment aca-env-demo \
    --yaml ./containerapp.yml

# Cập nhật app từ YAML
az containerapp update \
    --name ai-api \
    --resource-group rg-aca-demo \
    --yaml ./containerapp.yml
```

> **Quan trọng:** Khi dùng `--yaml`, **các flag CLI khác bị bỏ qua hoàn toàn**. Nếu bạn vừa có `--yaml` vừa có `--image`, chỉ YAML được dùng. YAML là source of truth tuyệt đối.

### Ví dụ YAML file cơ bản

```yaml
# containerapp.yml
properties:
  configuration:
    ingress:
      external: true
      targetPort: 80
  template:
    containers:
    - name: ai-api
      image: myregistry.azurecr.io/ai-api:v1
      resources:
        cpu: 0.5
        memory: 1Gi
```

YAML file này có thể lưu trong Git repo, review trong PR, và apply vào bất kỳ environment nào.

---

## Các deployment approach khác (tổng quan)

Trong project thực tế, team thường chuyển từ CLI-first sang pipeline có cấu trúc hơn:

| Approach | Dùng khi |
|---|---|
| `az containerapp up` | Prototype, first deploy, thử nghiệm nhanh |
| `az containerapp create/update` | CI/CD pipeline script, cần kiểm soát từng bước |
| YAML + CLI | Config phức tạp, cần code review, multi-environment |
| **Bicep / ARM** | Infrastructure as Code, deploy cả infrastructure + app |
| **GitHub Actions** | Continuous deployment tự động khi push code |
| **Azure Portal** | Unplanned change, emergency fix |

> Dù dùng approach nào, hiểu CLI và YAML vẫn quan trọng — chúng giúp bạn biết **property nào thay đổi khi deploy** và **nhìn vào đâu khi troubleshoot**.

---

## Tóm tắt luồng deploy cơ bản

```bash
# 1. Chuẩn bị CLI (một lần)
az login && az upgrade
az extension add --name containerapp --upgrade
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights

# 2. Tạo resource group và environment (một lần per environment)
az group create --name rg-aca-demo --location centralus
az containerapp env create --name aca-env-demo --resource-group rg-aca-demo --location centralus

# 3a. Deploy nhanh (prototype)
az containerapp up --name my-app --resource-group rg-aca-demo --environment aca-env-demo \
    --image mcr.microsoft.com/k8se/quickstart:latest --target-port 80 --ingress external

# 3b. Deploy explicit (production)
az containerapp create --name ai-api --resource-group rg-aca-demo --environment aca-env-demo \
    --image myregistry.azurecr.io/ai-api:v1 --ingress external --target-port 80

# 4. Update → tạo revision mới
az containerapp update --name ai-api --resource-group rg-aca-demo \
    --image myregistry.azurecr.io/ai-api:v2
```

---

## Bản chất bài này là gì?

**Một câu:** Ba cách deploy với trade-off ngược nhau: `up` (nhanh nhất, ít control nhất) → `create` (explicit) → YAML (chậm nhất để setup, nhiều control nhất, duy nhất phù hợp production).

### So sánh với Docker

| | Docker | ACA |
|---|---|---|
| Deploy nhanh | `docker run nginx` | `az containerapp up --image nginx` |
| Deploy tường minh | `docker run -p -e -v nginx` | `az containerapp create` với đầy đủ flags |
| Deploy từ config file | `docker-compose up` | `az containerapp create --yaml file.yml` |
| Cập nhật | `docker stop + docker run` new image | `az containerapp update` → tạo revision mới |

**Điểm khác biệt quan trọng với Docker:**
- `docker run` replace container cũ → không có lịch sử. `containerapp update` tạo **revision mới**, revision cũ vẫn còn → rollback nhanh.
- Khi dùng `--yaml`, tất cả CLI flag khác **bị bỏ qua** — không có hành vi này trong Docker.

---

## Checklist ghi nhớ cho AI-200

- [ ] Đăng ký resource providers trước khi dùng ACA: `Microsoft.App` và `Microsoft.OperationalInsights`
- [ ] `az containerapp up` = all-in-one, tự tạo resource, dùng cho prototype
- [ ] `az containerapp create` = explicit, cần environment có sẵn, dùng cho production
- [ ] Image repository name phải là **lowercase** — chữ hoa gây pull failure trông như auth error
- [ ] `--target-port` = port container lắng nghe bên trong
- [ ] `--ingress external` = public · `--ingress internal` = chỉ trong environment
- [ ] `az containerapp update` tạo **revision mới**, không modify in-place
- [ ] Revision là immutable snapshot — hỗ trợ rollback và traffic splitting
- [ ] Khi dùng `--yaml`, tất cả CLI flag khác **bị bỏ qua** — YAML là source of truth duy nhất
- [ ] YAML phù hợp cho production: reviewable qua PR, version controlled, repeatable

---

[← Bài 1](./aca-m1-bai1-explore-environments.md) · [🏠 Mục lục](../README.md) · [Bài 3 →](./aca-m1-bai3-runtime-settings.md)
