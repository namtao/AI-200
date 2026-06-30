# Bài 4 — Observe and Troubleshoot Containerized Apps

> Khoá: AI-200 · ACR — Deploy containers to Azure App Service

---

## Tổng quan

Deploy thành công chỉ là bước đầu. Trong production, bạn cần khả năng **quan sát** (observe) để biết app đang hoạt động thế nào, và **troubleshoot** nhanh khi có sự cố. Bài này bao gồm toàn bộ toolchain diagnostic của App Service:

```
Diagnostic tools
├── Container logs       → xem output từ stdout/stderr của container
├── Log stream           → xem log real-time
├── Kudu (SCM site)      → console nâng cao: env vars, file browser, diagnostic dump
├── Platform diagnostics → gửi log dài hạn đến Azure Monitor / Log Analytics
└── SSH access           → kết nối trực tiếp vào container đang chạy
```

---

## 1. Container Logs — Xem log của container

App Service tự động capture output từ **stdout** và **stderr** của container. Đây là nơi đầu tiên bạn nhìn vào khi có sự cố.

### Bật container logging

Mặc định logging chưa được bật. Cần enable trước:

```bash
az webapp log config \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --docker-container-logging filesystem
```

`filesystem` = lưu log vào App Service file system, available tại `/home/LogFiles/` và qua các tool diagnostic.

### Log bao gồm những gì?

| Loại log | Nội dung |
|---|---|
| Application output | Mọi thứ app của bạn `print()` / `log.info()` ra stdout |
| Error output | Exception stack trace, error message ra stderr |
| Framework logs | Web server startup (gunicorn, uvicorn...), request log |
| Platform messages | App Service thông báo về container lifecycle (start, stop, restart) |

### Cấu hình app để log có ích

App Service chỉ capture được những gì app **ghi ra stdout/stderr**. Nếu app dùng logging framework (Python `logging`, Node.js `winston`...), cần đảm bảo handler ghi ra console/stdout:

```python
import logging
import sys

# Ghi ra stdout để App Service capture được
logging.basicConfig(
    stream=sys.stdout,
    level=logging.INFO,
    format='%(asctime)s %(levelname)s %(message)s'
)
```

---

## 2. Log Stream — Xem log real-time

Log stream cho phép xem container output **trực tiếp khi nó xảy ra** — như `tail -f` nhưng qua network.

### Dùng CLI

```bash
az webapp log tail \
    --resource-group myResourceGroup \
    --name myDocumentProcessor
```

Log entries mới xuất hiện ngay khi container ghi ra. Nhấn `Ctrl+C` để dừng.

### Dùng Azure Portal

Vào web app → mục **Monitoring** → **Log stream**. Giao diện browser hiển thị log real-time, không cần cài CLI.

### Scaled-out app

Khi app có nhiều instance (scale out), log stream hiển thị output từ **tất cả instance**. Mỗi dòng log có **instance identifier** để bạn biết log đó đến từ instance nào — quan trọng khi debug lỗi chỉ xảy ra ở một vài instance cụ thể.

**Dùng log stream khi nào:**
- Debug startup failure — xem container crash ở bước nào
- Monitor app trong lúc test tải
- Quan sát behavior real-time khi demo hoặc rollout

---

## 3. Kudu / SCM Site — Console diagnostic nâng cao

Kudu (còn gọi là SCM site) là tool diagnostic mạnh nhất của App Service. Truy cập tại:

```
https://<app-name>.scm.azurewebsites.net
```

SCM site chạy như một **site riêng biệt** song song với app của bạn, yêu cầu xác thực bằng credentials có quyền manage web app.

### Tính năng quan trọng cho troubleshoot container

**Environment variables:**

Trang Environment hiển thị **toàn bộ environment variable** mà App Service inject vào container — bao gồm app settings của bạn lẫn system variable của Azure.

Truy cập trực tiếp: `https://<app-name>.scm.azurewebsites.net/Env`

Đây là cách nhanh nhất để verify:
- App setting đã được lưu đúng chưa
- Key Vault reference đã được resolve chưa
- Có typo trong tên setting không

**File system browser (Debug console):**

Cho phép duyệt file trong `/home` — nơi lưu persistent storage và log. Từ đây bạn có thể:
- Đọc file log tại `/home/LogFiles/`
- Kiểm tra file output mà app ghi ra `/home/output/`
- Download file để phân tích offline

**Diagnostic dump:**

Download file ZIP chứa log, config, và thông tin diagnostic. Hữu ích khi cần gửi cho support team hoặc phân tích offline.

### Giới hạn của Kudu

> **Quan trọng:** Kudu/SCM site **không phải** môi trường container của app. Nó chạy trong môi trường khác, nên:
> - Không thể browse file system bên trong container (chỉ browse được `/home`)
> - Không thể xem process đang chạy trong container
> - Không thể thực thi lệnh bên trong container

Để làm được những điều đó, cần dùng **SSH** (xem phần 5).

---

## 4. Platform Diagnostics — Log dài hạn với Azure Monitor

Container logs và log stream chỉ lưu tạm thời. Để **lưu log dài hạn**, phân tích pattern, tạo alert, và xây dashboard — cần gửi log đến **Azure Monitor** hoặc **Log Analytics workspace**.

### Cấu hình diagnostic settings

```bash
# Lấy resource ID của web app
resourceId=$(az webapp show \
    -g myResourceGroup \
    -n myDocumentProcessor \
    --query id -o tsv)

# Lấy ID của Log Analytics workspace
workspaceId=$(az monitor log-analytics workspace show \
    -g myResourceGroup \
    -n myLogAnalyticsWorkspace \
    --query id -o tsv)

# Tạo diagnostic settings
az monitor diagnostic-settings create \
    --resource "$resourceId" \
    --name myDiagnosticSettings \
    --workspace "$workspaceId" \
    --logs '[{"category":"AppServiceConsoleLogs","enabled":true},{"category":"AppServiceHTTPLogs","enabled":true}]'
```

### Các log category có sẵn

| Category | Nội dung |
|---|---|
| `AppServiceConsoleLogs` | Container stdout và stderr — log chính của app |
| `AppServiceHTTPLogs` | HTTP request/response: method, URL, status code, latency |
| `AppServicePlatformLogs` | Container lifecycle events: start, stop, crash, restart |
| `AppServiceAppLogs` | Application-level logs (cần cấu hình thêm trong app) |

### Query log trong Log Analytics

Sau khi log chảy vào Log Analytics, dùng **Kusto Query Language (KQL)** để phân tích:

```kusto
// Xem tất cả error trong 1 giờ qua
AppServiceConsoleLogs
| where Level == "Error"
| where TimeGenerated > ago(1h)
| project TimeGenerated, ResultDescription
| order by TimeGenerated desc
```

```kusto
// Xem HTTP request chậm (> 2 giây)
AppServiceHTTPLogs
| where TimeTaken > 2000
| project TimeGenerated, CsMethod, CsUriStem, ScStatus, TimeTaken
| order by TimeTaken desc
```

Log Analytics cho phép tạo alert tự động (ví dụ: alert khi error rate tăng đột biến) và build dashboard để monitor production.

---

## 5. SSH Access — Kết nối trực tiếp vào container

SSH cho phép bạn mở terminal **bên trong container đang chạy** — cách mạnh nhất để debug khi log không đủ thông tin.

### Yêu cầu: Cấu hình SSH trong container image

App Service SSH không tự động có sẵn — bạn phải **cấu hình trong Dockerfile**:

```dockerfile
# Cài OpenSSH server
RUN apt-get update && apt-get install -y openssh-server \
    && echo "root:Docker!" | chpasswd

# Copy SSH config (cần tạo file sshd_config riêng)
COPY sshd_config /etc/ssh/

# Expose port app + port SSH (2222)
EXPOSE 8000 2222

# Start SSH daemon + app cùng lúc
CMD ["/bin/bash", "-c", "service ssh start && gunicorn app:application"]
```

**Các yêu cầu bắt buộc:**
- Cài `openssh-server`
- SSH phải lắng nghe trên **port 2222** (không phải 22)
- Root password phải là **`Docker!`** (yêu cầu của App Service)
- Start SSH daemon cùng lúc với app

### Truy cập SSH

Sau khi image được cấu hình đúng, vào Azure Portal → web app → mục **Development Tools** → **SSH**. Portal mở terminal session trực tiếp vào container.

Từ SSH session, bạn có thể:
- Kiểm tra environment variable thực tế: `env | grep API_KEY`
- Xem process đang chạy: `ps aux`
- Kiểm tra network connectivity: `curl http://internal-service`
- Đọc file log trong container
- Debug runtime issue trực tiếp

---

## 6. Common Issues và Solutions

### Container fails to start

**Triệu chứng:** URL của app trả về error page, container logs có startup failure.

**Cách diagnose:**
1. Dùng log stream (`az webapp log tail`) để xem container crash ở đâu
2. Kiểm tra image có tồn tại trong registry không
3. Xác nhận credentials (managed identity có role AcrPull chưa)
4. Thử chạy container local với cùng environment variable để reproduce

**Nguyên nhân thường gặp:**
- App đọc environment variable bắt buộc lúc startup nhưng variable chưa được set
- `WEBSITES_PORT` không khớp với port container thực sự lắng nghe
- App crash do thiếu dependency hoặc config file
- Image bị corrupt hoặc tag không tồn tại

---

### 404 responses sau khi deploy thành công

**Triệu chứng:** Container start được (không thấy crash log), nhưng mọi request đều trả về 404.

**Cách diagnose:**
1. Verify `WEBSITES_PORT` khớp với port app lắng nghe
2. Kiểm tra app bind vào `0.0.0.0` (tất cả interface), không phải `localhost`
3. Xác nhận app có route cho path được gọi

**Nguyên nhân thường gặp:**

**App bind vào localhost thay vì 0.0.0.0:**

```python
# Sai — chỉ nhận request từ trong container, không nhận từ App Service load balancer
app.run(host='localhost', port=8000)

# Đúng — nhận request từ tất cả network interface
app.run(host='0.0.0.0', port=8000)
```

**Port mismatch:**
```
Container lắng nghe: 8000
WEBSITES_PORT được set: 5000  ← App Service forward đến port sai → 404
```

---

### Missing environment variables

**Triệu chứng:** App log báo lỗi về missing config, `None` hoặc `undefined` khi đọc env var.

**Cách diagnose:**
1. Vào Kudu → Environment page → tìm tên variable
2. Kiểm tra setting đã được save chưa (Portal có nút Save riêng)
3. Kiểm tra typo trong tên setting

**Nguyên nhân thường gặp:**
- Quên nhấn Save sau khi edit trong Portal
- Typo: `STORAGE_ACCOUNT_NAMEE` thay vì `STORAGE_ACCOUNT_NAME`
- App đọc env var trước khi App Service kịp inject (hiếm gặp, thường với app start rất nhanh)

---

### Slow cold starts

**Triệu chứng:** Request đầu tiên sau khi app idle tốn nhiều giây, các request sau bình thường.

**Cách diagnose:**
1. Kiểm tra kích thước image: `docker images <image-name>`
2. Xem log startup time — app mất bao lâu từ khi start đến khi ready
3. Kiểm tra always-on có được bật chưa

**Giải pháp:**
- **Bật always-on** (Basic tier+) để app không bao giờ idle
- **Giảm kích thước image** bằng cách dùng base image nhỏ hơn hoặc multi-stage build
- **Defer heavy initialization** — không load toàn bộ ML model lúc startup, lazy load khi cần
- **Optimize layer order** trong Dockerfile để tận dụng cache

---

## Tóm tắt — Chọn tool phù hợp với tình huống

| Tình huống | Tool phù hợp |
|---|---|
| App vừa deploy, cần debug ngay | Log stream (`az webapp log tail`) |
| Kiểm tra env var có được inject đúng không | Kudu → Environment page |
| Xem log file đã lưu | Kudu → Debug console → `/home/LogFiles/` |
| Phân tích pattern lỗi theo thời gian | Log Analytics + KQL query |
| Tạo alert khi error rate tăng | Log Analytics + Azure Monitor alert |
| Debug runtime issue cần thực thi lệnh trong container | SSH (cần cấu hình trong Dockerfile) |
| Cần share diagnostic info với support team | Kudu → Diagnostic dump |

---

## Bản chất bài này là gì?

**Một câu:** Bài này là "bạn đã mất `docker exec` khi lên PaaS — đây là những gì thay thế nó."

Với Docker thuần, khi có sự cố bạn có full control: `docker logs`, `docker exec -it bash`, `docker inspect`. Trên App Service, container chạy trên infrastructure Azure quản lý — bạn không có SSH mặc định, không có `docker exec`. Thay vào đó App Service cung cấp một bộ tool riêng.

### So sánh toolchain debug: Docker vs App Service

| Tình huống debug | Docker thuần | App Service |
|---|---|---|
| Xem log container | `docker logs <container>` | `az webapp log tail` / Log stream |
| Xem log real-time | `docker logs -f` | `az webapp log tail` (tương đương) |
| Xem environment variable | `docker inspect <container>` | Kudu → `/Env` page |
| Vào terminal trong container | `docker exec -it <container> bash` | SSH — nhưng phải cài sẵn trong Dockerfile |
| Browse file system trong container | `docker exec` → `ls` | Kudu file browser — chỉ xem được `/home`, không vào được container |
| Log dài hạn / alerting | Docker logging driver (Splunk, ELK, Fluentd...) | Azure Monitor + Log Analytics + KQL |
| Platform events (start/stop/crash) | `docker events` | `AppServicePlatformLogs` trong Log Analytics |

**3 điểm khác biệt quan trọng nhất so với Docker:**

1. **SSH phải được baked vào image — không có sau-sự-cố:** `docker exec` luôn có sẵn miễn là container đang chạy. SSH vào App Service yêu cầu cài `openssh-server` trong Dockerfile trước khi build. Nếu quên, khi sự cố xảy ra không còn cách vào container. Đây là lý do SSH config thường được thêm vào production image từ đầu, dù hiếm khi dùng.

2. **Kudu chạy ngoài container — không phải trong container:** Dễ nhầm Kudu là "console của container". Thực ra Kudu là một site riêng biệt chạy song song. `docker inspect` cho bạn config của container từ góc nhìn Docker daemon. Kudu `/Env` cho bạn thấy env var *được inject vào* container — nhưng Kudu không thể browse file system bên trong container, chỉ browse được `/home` (shared volume).

3. **Log phải ra stdout/stderr — cả Docker lẫn App Service đều chỉ capture đây:** Nếu app ghi log ra file (ví dụ `/var/log/app.log`) thay vì stdout, `docker logs` cũng trắng, `az webapp log tail` cũng trắng. Đây là "gotcha" phổ biến với app legacy hoặc framework có default ghi ra file.

---

## Checklist ghi nhớ cho AI-200

- [ ] Bật container logging: `az webapp log config --docker-container-logging filesystem`
- [ ] Log được lưu tại `/home/LogFiles/` (khi bật persistent storage)
- [ ] App phải ghi ra **stdout/stderr** để App Service capture được
- [ ] Log stream real-time: `az webapp log tail` hoặc qua Portal → Log stream
- [ ] Log stream scaled-out app hiển thị từ **tất cả instance**, có instance identifier
- [ ] Kudu URL: `https://<app-name>.scm.azurewebsites.net`
- [ ] Kudu Environment page: verify env var tại `https://<app-name>.scm.azurewebsites.net/Env`
- [ ] Kudu **không phải** môi trường container — không browse được file system bên trong container
- [ ] Log dài hạn: gửi đến **Log Analytics** qua `az monitor diagnostic-settings create`
- [ ] 4 log categories chính: `AppServiceConsoleLogs`, `AppServiceHTTPLogs`, `AppServicePlatformLogs`, `AppServiceAppLogs`
- [ ] SSH trong container yêu cầu: cài `openssh-server`, listen port **2222**, root password **`Docker!`**
- [ ] **Container fails to start** → check log stream + verify WEBSITES_PORT + verify AcrPull role
- [ ] **404 sau deploy** → check app bind `0.0.0.0` (không phải `localhost`) + check WEBSITES_PORT
- [ ] **Missing env var** → check Kudu Environment page + check typo + check đã Save chưa
- [ ] **Slow cold start** → bật always-on + giảm image size + defer heavy initialization

---

[← Bài 3](./acr-m2-bai3-configure-app-settings.md) · [🏠 Mục lục](../README.md) · [Assessment →](./acr-m2-module-assessment.md)
