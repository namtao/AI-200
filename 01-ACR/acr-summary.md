# Tổng hợp kiến thức — 01-ACR (2 Modules)

> AI-200 · Module 1: Store and Manage Containers in ACR · Module 2: Deploy Containers to Azure App Service

---

## Bức tranh tổng thể

```
Developer
    │
    ▼
[Module 1] Build image → Push lên ACR
    │       az acr build  myregistry.azurecr.io/app:v1
    │
    ▼
[Module 2] ACR → App Service pull image → Web app chạy
               az webapp create --container-image-name ...
```

**Module 1** = quản lý image (lưu trữ, build, versioning)  
**Module 2** = chạy image thành web app (deploy, cấu hình, debug)

---

## MODULE 1 — Azure Container Registry

### Cấu trúc 3 tầng

```
Registry      myregistry.azurecr.io
└── Repository    inference-api
    └── Artifact  :v1.0.0  (= tag + layers + manifest)
```

- **Registry URL:** `<tên>.azurecr.io`
- **Namespace** dùng `/` → tổ chức và phân quyền: `production/app`, `ml-team/model`
- ACR lưu được: Docker image, Helm chart, OCI artifact

### Tags, Layers, Manifests

| Thành phần | Đặc điểm | Dùng khi |
|---|---|---|
| **Tag** | Mutable — có thể bị ghi đè | Dev, staging, base image |
| **Digest** (`sha256:...`) | Immutable — không bao giờ thay đổi | Production deployment |
| **Layer** | Deduplication — nhiều image chia sẻ layer chung | Tiết kiệm storage |

> **Exam trap:** Tag `v1.2.0` vẫn là mutable về mặt kỹ thuật. Chỉ **digest** mới đảm bảo immutability tuyệt đối.

### ACR Tiers

| Tier | Điểm khác biệt |
|---|---|
| Basic | Học tập, dev — storage/throughput giới hạn |
| Standard | Production thông thường |
| **Premium** | **Geo-replication + Private endpoint + Content trust + Retention policy** |

> **Exam trap:** Geo-replication và retention policy **chỉ có Premium**.

### ACR Tasks — 3 loại

| Loại | Trigger | Dùng khi |
|---|---|---|
| **Quick Task** | Thủ công (`az acr build`) | Build on-demand, không cần setup |
| **Triggered Task** | Git commit / Base image update / Schedule | CI/CD automation |
| **Multi-step Task** | Bất kỳ trigger trên | Build → Test → Push trong 1 workflow (file YAML) |

**Base image trigger** = killer feature của ACR Tasks — tự động rebuild khi `FROM <base-image>` được cập nhật. Hoạt động tốt nhất khi cả hai image **cùng registry**.

**Scheduled trigger** dùng **cron expression** — `"0 0 * * *"` = hàng ngày 00:00 UTC.

**Build context** có thể là: local directory, Git URL, remote tarball.  
**`{{.Run.ID}}`** = biến tự tạo unique tag mỗi build (dùng thay `latest`).

### Tagging Strategy

**Stable tag** (bị ghi đè): `latest`, `v1`, `nightly` → base image, dev environment  
**Unique tag** (không tái sử dụng): `v1.2.0-abc123f` → production bắt buộc

**Semantic versioning:**
- `MAJOR` — breaking change
- `MINOR` — tính năng mới, backward compatible
- `PATCH` — bug fix, security patch

**Vấn đề của `latest` trong production:** Node A pull lúc 9h, Node B pull lúc 14h → hai node nhận image khác nhau nếu ai đó push build mới → cluster chạy 2 version cùng lúc.

### Image Lifecycle Management

```bash
# Lock image production
az acr repository update --name reg --image app:v1.2.0 --write-enabled false

# Cleanup untagged images (cũ hơn 30 ngày)
az acr run --registry reg \
  --cmd "acr purge --filter 'app:.*' --untagged --ago 30d" /dev/null

# Scheduled weekly cleanup
az acr task create --registry reg --name cleanup \
  --cmd "acr purge --filter '.*:.*' --untagged --ago 7d" \
  --schedule "0 0 * * 0" --context /dev/null
```

**Retention policy** (tự động, set-and-forget) = **Premium only**.

---

## MODULE 2 — Deploy Containers to Azure App Service

### Bản chất

App Service = "Docker host được quản lý" — bạn cung cấp image, Azure lo server, load balancing, TLS, scaling.

```
Docker thuần:   docker run -p -v -e image    (bạn kiểm soát mọi thứ)
App Service:    khai báo image → Azure lo    (đổi control lấy simplicity)
```

### Deploy

**Yêu cầu:** Container phải chạy trên **Linux** (Web App for Containers không hỗ trợ Windows).

**Image sources:**
- **ACR** (khuyến nghị) — tích hợp managed identity, không cần credential
- **Other registries** (Docker Hub, GHCR, self-hosted) — cần hỗ trợ Docker Registry HTTP API V2

**ACR Authentication:**

| Cách | Bảo mật | Dùng khi |
|---|---|---|
| **Managed Identity** | ✅ Production | Không bao giờ thấy password |
| **Admin credentials** | ⚠️ Dev only | Lưu password trong config |

- **System-assigned identity** — gắn với vòng đời web app, xoá app là mất
- **User-assigned identity** — tồn tại độc lập, dùng được cho nhiều app

Managed identity cần role **`AcrPull`** trên registry.

```bash
# Deploy từ ACR
az webapp create \
    --resource-group rg --plan plan --name myapp \
    --container-image-name myregistry.azurecr.io/app:v1

# Cập nhật version mới
az webapp config container set \
    --resource-group rg --name myapp \
    --container-image-name myregistry.azurecr.io/app:v2
```

**App Service KHÔNG tự detect** khi push image mới vào cùng tag → phải restart thủ công hoặc dùng webhook.

**Continuous deployment:**
```bash
az webapp deployment container config --resource-group rg --name myapp --enable-cd true
# → nhận webhook URL → cấu hình vào ACR webhook
az acr webhook create --registry reg --name hook --uri <webhook-url> --actions push
```

### Configure Runtime (5 settings)

| Setting | Docker tương đương | Mặc định |
|---|---|---|
| Startup command | `docker run <image> <cmd>` | CMD trong Dockerfile |
| `WEBSITES_PORT` | `-p host:container` (1 chiều) | Tự detect 80/8080 |
| `WEBSITES_ENABLE_APP_SERVICE_STORAGE=true` | `-v path:/home` | **Tắt** — file system ephemeral |
| `--always-on true` | N/A (PaaS-specific) | Tắt — app idle sau 20 phút |
| Health check (`/health`) | `HEALTHCHECK` Dockerfile | Không có |

**Startup command** override `CMD`, không override `ENTRYPOINT`.  
Chạy nhiều lệnh: `"/bin/bash -c 'python migrate.py && gunicorn app:application'"`.

**TLS termination:** App Service decrypt HTTPS trước, container chỉ nhận HTTP — không cần cấu hình SSL trong container.

**App Service chỉ expose 1 port** — cần nhiều port → dùng ACA hoặc AKS.

**Always-on** yêu cầu **Basic tier trở lên**.

**Health check:** probe HTTP mỗi phút, fail 10 lần → loại khỏi load balancer. Response 200 = healthy. Thay đổi config → **restart app**.

**Cold start** tốn thời gian vì: pull image layers, khởi động process, load ML model. Bật always-on để tránh.

### Configure App Settings

**App settings = env var inject lúc runtime** — same image, khác config theo environment.

```bash
az webapp config appsettings set --settings KEY=VALUE KEY2=VALUE2
```

- Tên setting: chữ, số, `_` . Nested config .NET dùng `__` thay `:` trên Linux.
- Tất cả app settings **encrypted at rest**.

**Connection strings** → auto thêm prefix (`SQLAZURECONNSTR_`, `POSTGRESQLCONNSTR_`,...) → chỉ hữu ích cho .NET. Python/Node.js → dùng app settings thông thường.

**Bulk edit:** export `appsettings list --output json > file.json` → import `appsettings set --settings @file.json`

**Slot settings** — gắn với slot, không swap khi swap deployment slot:
```bash
az webapp config appsettings set --slot staging --slot-settings ENVIRONMENT=staging API_URL=https://api-staging.example.com
```
Dùng cho: environment name, env-specific URL, feature flags, log level.

**Key Vault references** — secret không bao giờ chạm App Service:
```bash
--settings API_KEY="@Microsoft.KeyVault(SecretUri=https://vault.vault.azure.net/secrets/key)"
```
Yêu cầu: managed identity có role **`Key Vault Secrets User`**. Không có version → luôn lấy latest, auto-refresh 24h. Restart → force refetch ngay.

### Observe & Troubleshoot

| Tình huống | Tool |
|---|---|
| Debug startup failure, log real-time | `az webapp log tail` / Portal Log stream |
| Verify env var inject đúng chưa | Kudu `https://<app>.scm.azurewebsites.net/Env` |
| Browse file log `/home/LogFiles/` | Kudu Debug console |
| Log dài hạn, alert, dashboard | Azure Monitor + Log Analytics (KQL) |
| Thực thi lệnh trong container | SSH (phải cấu hình trong Dockerfile trước) |

**Bật logging:** `az webapp log config --docker-container-logging filesystem`

App phải ghi ra **stdout/stderr** — App Service (và Docker) chỉ capture stdout/stderr.

**Log stream scaled-out:** hiển thị từ tất cả instance, có instance identifier.

**Kudu KHÔNG phải môi trường container** — không browse được file system bên trong container.

**SSH setup trong Dockerfile** (phải làm từ trước):
- Cài `openssh-server`
- Listen port **2222** (không phải 22)
- Root password: **`Docker!`**

**4 log categories chính:** `AppServiceConsoleLogs`, `AppServiceHTTPLogs`, `AppServicePlatformLogs`, `AppServiceAppLogs`

### Common Issues

| Triệu chứng | Nguyên nhân phổ biến | Fix |
|---|---|---|
| Container không start | Missing env var / WEBSITES_PORT sai / AcrPull thiếu | Check log stream |
| 404 sau deploy thành công | App bind `localhost` thay vì `0.0.0.0` / port mismatch | Check binding + WEBSITES_PORT |
| Missing env var | Quên nhấn Save / typo | Kudu /Env page |
| Slow cold start | Image lớn / load ML model / always-on tắt | Bật always-on + giảm image size |

---

## CLI Commands Quick Reference

```bash
# === ACR ===
az acr build --registry reg --image app:v1 .                    # Quick build
az acr task create --registry reg --name t --image app:{{.Run.ID}} \
  --context https://github.com/org/repo.git#main \
  --file Dockerfile --git-access-token $PAT                     # Triggered task
az acr run --registry reg --cmd 'app:v1 python --version' /dev/null  # Run container
az acr repository update --name reg --image app:v1 --write-enabled false  # Lock
az acr webhook create --registry reg --name hook --uri <url> --actions push

# === App Service — Deploy ===
az webapp create --resource-group rg --plan plan --name app \
  --container-image-name reg.azurecr.io/img:v1
az webapp config container set --resource-group rg --name app \
  --container-image-name reg.azurecr.io/img:v2
az webapp deployment container config --resource-group rg --name app --enable-cd true
az webapp show --resource-group rg --name app --query defaultHostName --output tsv

# === App Service — Runtime ===
az webapp config set --resource-group rg --name app \
  --startup-file "gunicorn --bind=0.0.0.0:8000 app:app" --always-on true
az webapp config appsettings set --resource-group rg --name app \
  --settings WEBSITES_PORT=8000 WEBSITES_ENABLE_APP_SERVICE_STORAGE=true
az webapp config set --resource-group rg --name app \
  --generic-configurations '{"healthCheckPath": "/health"}'

# === App Service — Settings ===
az webapp config appsettings set --settings KEY=VALUE
az webapp config appsettings list --output json > settings.json
az webapp config appsettings set --settings @settings.json
az webapp config appsettings set --slot staging --slot-settings ENV=staging
az webapp config connection-string set --connection-string-type SQLAzure \
  --settings DefaultConn="Server=..."

# === App Service — Diagnostics ===
az webapp log config --docker-container-logging filesystem
az webapp log tail --resource-group rg --name app
# Kudu: https://<app>.scm.azurewebsites.net/Env
```

---

## Exam Traps — Những điểm hay nhầm

1. **Tag ≠ immutable.** Dù là `v1.2.0` cũng có thể bị push đè. Chỉ **digest** là immutable tuyệt đối.

2. **Geo-replication + Content trust + Retention policy = Premium only.** Basic/Standard không có.

3. **Startup command override CMD, không override ENTRYPOINT.**

4. **WEBSITES_PORT = port của container**, không phải port bên ngoài (App Service tự lo 80/443).

5. **Persistent storage TẮT mặc định** trên Linux custom container. Phải bật `WEBSITES_ENABLE_APP_SERVICE_STORAGE=true`.

6. **Always-on cần Basic tier trở lên.** Free/Shared không có.

7. **Kudu không phải container** — không vào được file system trong container, chỉ xem `/home`.

8. **Health check thay đổi → restart app.** Làm trong maintenance window.

9. **Connection strings với Python/Node.js** → dùng app settings là đủ, không cần nhớ prefix.

10. **SSH vào container phải cấu hình sẵn trong Dockerfile** — port 2222, password `Docker!`. Không thể thêm sau khi sự cố xảy ra.

11. **App Service KHÔNG tự detect** khi push image mới vào cùng tag — phải restart thủ công hoặc webhook.

12. **Base image trigger** hoạt động tốt nhất khi cả hai image cùng ACR registry, không phải Docker Hub.

---

## Assessment Summary

### Module 1 — ACR
| Câu | Tình huống | Đáp án |
|---|---|---|
| 1 | Build không nhất quán giữa các dev | ACR Tasks quick build |
| 2 | Đảm bảo mọi k8s node chạy đúng cùng image | Manifest digest |
| 3 | Tự động rebuild khi base image (PyTorch) có patch | Base image update trigger |
| 4 | Traceability đến commit + rollback | Unique tag với Git commit SHA |
| 5 | Ngăn xoá image production | `--write-enabled false` |

### Module 2 — App Service
| Câu | Tình huống | Đáp án |
|---|---|---|
| 1 | Container lắng nghe port 8000, connection error | `WEBSITES_PORT=8000` |
| 2 | File output mất sau restart | `WEBSITES_ENABLE_APP_SERVICE_STORAGE=true` + ghi vào `/home` |
| 3 | API URL không swap khi swap slot | Đánh dấu là slot setting |
| 4 | Health check fail dù app có `/healthz` | Path mismatch — sửa health check path thành `/healthz` |
| 5 | Verify env var inject đúng | Kudu Environment page |

---

[← Assessment](./acr-m2-module-assessment.md) · [🏠 Mục lục](../README.md) · [Mục lục →](../README.md)
