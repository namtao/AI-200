# Bài 2 — Configure Container Runtime Behavior

> Khoá: AI-200 · ACR — Deploy containers to Azure App Service

---

## Tổng quan

Sau khi deploy container lên App Service, bạn cần cấu hình cách App Service **chạy** container đó. Bài này bao gồm 5 nhóm cấu hình:

```
Runtime behavior
├── Startup commands    → chỉ định lệnh khởi động container
├── Port configuration  → cho App Service biết container lắng nghe port nào
├── Persistent storage  → dữ liệu có tồn tại qua restart không?
├── Always-on           → ngăn cold start khi app bị idle
└── Health checks       → tự động phát hiện và xử lý instance không healthy
```

---

## 1. Startup Commands — Lệnh khởi động

Mặc định, App Service chạy container theo `ENTRYPOINT` và `CMD` được định nghĩa trong Dockerfile. Tuy nhiên bạn có thể **override lệnh khởi động** mà không cần rebuild image.

### Tại sao cần override startup command?

- Truyền runtime argument khác nhau tùy environment (dev vs production)
- Chạy database migration trước khi start app
- Điều chỉnh số worker theo tier của App Service plan
- Start nhiều process trong cùng một container

### Cấu hình startup command

```bash
az webapp config set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --startup-file "gunicorn --bind=0.0.0.0:8000 --workers=4 app:application"
```

Lệnh này override `CMD` trong Dockerfile. `ENTRYPOINT` vẫn giữ nguyên.

> **Lưu ý quan trọng:** `--startup-file` override `CMD`, không override `ENTRYPOINT`. Nếu Dockerfile của bạn dùng ENTRYPOINT dạng exec form (`ENTRYPOINT ["python"]`) thì startup command sẽ được truyền vào như argument. Nếu không có ENTRYPOINT, startup command chạy trực tiếp.

### Startup command có shell processing

Khi cần chạy nhiều lệnh liên tiếp (ví dụ: migrate rồi mới start app), dùng shell để xử lý:

```bash
az webapp config set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --startup-file "/bin/bash -c 'python migrate.py && gunicorn app:application'"
```

`/bin/bash -c '...'` cho phép dùng các shell operator như `&&` (chạy lệnh tiếp theo chỉ khi lệnh trước thành công), `;` (chạy tuần tự bất kể kết quả), hay `||` (chạy lệnh tiếp theo nếu lệnh trước fail).

---

## 2. Port Configuration — Cấu hình port

App Service cần biết **container của bạn đang lắng nghe ở port nào** để forward HTTP traffic vào đúng chỗ.

### Cơ chế mặc định

App Service tự động route traffic nếu container lắng nghe trên **port 80 hoặc 8080** — không cần cấu hình gì thêm.

Nếu container lắng nghe trên port khác, bạn phải khai báo qua app setting `WEBSITES_PORT`:

```bash
az webapp config appsettings set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --settings WEBSITES_PORT=8000
```

### TLS termination

Một điểm quan trọng cần hiểu: App Service **xử lý TLS trước khi traffic vào container**. Nghĩa là:

- Client kết nối HTTPS → App Service nhận và decrypt
- Container nhận HTTP traffic (không phải HTTPS)
- Container không cần cấu hình TLS/SSL certificate

Đây gọi là "TLS termination at the load balancer" — pattern phổ biến trong cloud architecture.

### App Service chỉ expose một port

App Service chỉ hỗ trợ expose **một port duy nhất** cho HTTP requests từ custom container. Nếu app của bạn cần expose nhiều port (ví dụ: HTTP + gRPC), cần kiến trúc khác (Azure Container Apps hoặc AKS).

### Port mặc định theo framework

| Framework | Default Port | Cần cấu hình |
|---|---|---|
| Node.js (Express) | 3000 | `WEBSITES_PORT=3000` |
| Python (Gunicorn) | 8000 | `WEBSITES_PORT=8000` |
| Java (Spring Boot) | 8080 | Không cần (tự detect) |
| ASP.NET Core | 80 | Không cần (tự detect) |

---

## 3. Persistent Storage — Lưu trữ bền vững

### Vấn đề: Container file system là ephemeral

Mặc định, toàn bộ dữ liệu bạn ghi vào file system của container sẽ **mất khi app restart** hoặc khi Azure di chuyển app sang infrastructure khác. Đây là behavior chuẩn của container — container được thiết kế để stateless.

Điều này có nghĩa là:
- Log files bị xoá khi restart
- File upload của user bị mất khi restart
- Cache bị xoá khi restart
- Bất kỳ file nào được tạo lúc runtime đều biến mất

### Giải pháp: Bật persistent storage tại `/home`

App Service cho phép mount persistent storage tại thư mục `/home`. Mặc định tính năng này **bị tắt** cho Linux custom container.

```bash
az webapp config appsettings set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --settings WEBSITES_ENABLE_APP_SERVICE_STORAGE=true
```

Sau khi bật:
- Mọi dữ liệu ghi vào `/home` tồn tại qua restart
- Tất cả instance trong scaled-out app **chia sẻ cùng một `/home`** — mọi instance đều thấy cùng nội dung
- `/home/LogFiles` lưu container log và application log

### Lưu ý khi dùng persistent storage

**Shared storage giữa các instance:** Khi scale out 3 instance, cả 3 đều ghi vào cùng `/home`. Nếu app của bạn không được thiết kế để xử lý concurrent writes, có thể xảy ra race condition hoặc file corruption.

**Storage quota:** Dung lượng phụ thuộc vào App Service plan và được **chia sẻ giữa tất cả app trong cùng plan**. Không phù hợp cho ứng dụng cần storage lớn hoặc I/O cao.

**Khi nào cần giải pháp khác:** Nếu app cần storage lớn, I/O cao, hoặc cần isolate storage giữa các instance — dùng **Azure Blob Storage** (qua SDK) hoặc **mount Azure Storage** như một extra volume.

Cấu hình app ghi dữ liệu vào `/home` hoặc subdirectory:

```
/home/output/        # processed files
/home/uploads/       # temporary uploads
/home/LogFiles/      # logs (tự động)
```

---

## 4. Always-On — Ngăn cold start

### Vấn đề: Cold start

App Service có cơ chế tiết kiệm tài nguyên: sau khoảng **20 phút không có traffic**, app sẽ bị "ngủ" (idle). Request tiếp theo phải chờ app khởi động lại — gọi là **cold start**.

Cold start với container có thể tốn nhiều thời gian vì:
- Pull image (nếu layer bị evict khỏi cache)
- Khởi động process bên trong container
- Kết nối lại database, cache, external services
- Load AI model vào memory (với AI inference service, đây có thể tốn hàng chục giây)

### Giải pháp: Always-on

```bash
az webapp config set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --always-on true
```

Khi bật, App Service gửi periodic request đến app để giữ nó luôn warm — app không bao giờ bị idle.

**Yêu cầu tier:** Always-on chỉ có từ **Basic pricing tier trở lên**. Free và Shared tier không hỗ trợ.

### Khi nào nên bật Always-on?

- Production app có SLA về response time
- App có startup time dài (container image lớn, load ML model...)
- App duy trì background process hoặc persistent connection (WebSocket, message queue consumer)
- App phục vụ user có thể truy cập bất kỳ lúc nào trong ngày

### Khi nào không cần Always-on?

- Dev/staging environment không cần SLA
- App chỉ được gọi theo lịch cố định (cron job) — có thể chấp nhận cold start vài giây
- Muốn tiết kiệm chi phí cho app ít traffic

---

## 5. Health Checks — Kiểm tra sức khỏe container

### Vấn đề: App chạy nhưng không serve request được

Container có thể ở trạng thái "running" (process đang chạy) nhưng thực ra không serve request được — ví dụ: database connection bị dropped, memory exhausted, internal deadlock. Load balancer không biết điều này và vẫn route traffic vào instance đó.

Health check giải quyết vấn đề này bằng cách chủ động probe app định kỳ.

### Cấu hình health check

```bash
az webapp config set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --generic-configurations '{"healthCheckPath": "/health"}'
```

App Service sẽ gửi HTTP GET đến `/health` mỗi phút. Response HTTP 200 = healthy, response khác hoặc timeout = unhealthy.

**Cơ chế xử lý instance unhealthy:**
- Sau 10 lần fail liên tiếp → App Service loại instance đó ra khỏi load balancer rotation (không route traffic vào nữa)
- Nếu instance tiếp tục unhealthy sau thời gian dài → App Service thay thế instance đó bằng instance mới

### Implement health endpoint trong app

**Đơn giản — chỉ kiểm tra app có respond không:**

```python
@app.route('/health')
def health_check():
    return {'status': 'healthy'}, 200
```

**Đầy đủ hơn — kiểm tra các dependency:**

```python
@app.route('/health')
def health_check():
    try:
        # Kiểm tra database connection
        db.execute('SELECT 1')
        # Kiểm tra storage access
        storage.list_containers()
        return {'status': 'healthy'}, 200
    except Exception as e:
        return {'status': 'unhealthy', 'error': str(e)}, 503
```

Health endpoint nên kiểm tra những gì app thực sự cần để serve request:
- Database connectivity
- Cache connectivity (Redis...)
- External API availability
- Disk space / memory đủ không
- ML model đã load chưa (với AI inference service)

### Lưu ý quan trọng

> **Health check configuration changes restart app.** Áp dụng thay đổi health check trong maintenance window, không làm giữa production traffic.

Health endpoint phải **nhẹ và nhanh** — nó được gọi mỗi phút. Không thực hiện các operation nặng (query phức tạp, full model inference) trong health check.

---

## Tổng hợp — Tất cả settings trong một chỗ

```bash
# 1. Startup command (override CMD trong Dockerfile)
az webapp config set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --startup-file "gunicorn --bind=0.0.0.0:8000 --workers=4 app:application"

# 2. Port (nếu container không dùng 80/8080)
az webapp config appsettings set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --settings WEBSITES_PORT=8000

# 3. Persistent storage
az webapp config appsettings set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --settings WEBSITES_ENABLE_APP_SERVICE_STORAGE=true

# 4. Always-on (Basic tier+)
az webapp config set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --always-on true

# 5. Health check
az webapp config set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --generic-configurations '{"healthCheckPath": "/health"}'
```

---

## Bản chất bài này là gì?

**Một câu:** Bài này giải quyết những gì Docker *không làm* khi bạn giao container cho một managed platform.

Khi chạy Docker thuần, bạn kiểm soát mọi thứ qua `docker run`: port mapping, volume, startup command, health check — tất cả khai báo thẳng tại dòng lệnh. Khi chạy trên App Service (PaaS), bạn không có `docker run`. Thay vào đó, App Service cần bạn *khai báo ý định* qua settings, rồi nó tự orchestrate.

### So sánh Docker thuần vs App Service

| Tính năng | Docker thuần | App Service |
|---|---|---|
| Override startup command | `docker run <image> <cmd>` | `az webapp config set --startup-file "..."` |
| Khai báo port | `-p 8080:3000` (map 2 chiều) | `WEBSITES_PORT=3000` (chỉ nói port container) |
| Persistent storage | `-v /host/path:/container/path` | `WEBSITES_ENABLE_APP_SERVICE_STORAGE=true` → mount sẵn `/home` |
| App "ngủ" khi idle | Không xảy ra — bạn kiểm soát host | Xảy ra sau 20 phút → cần `--always-on true` |
| Health check | `HEALTHCHECK` trong Dockerfile (chạy bên trong container) | App Service probe HTTP từ ngoài vào mỗi phút |
| TLS/HTTPS | Tự cấu hình reverse proxy + cert | App Service terminate TLS trước — container chỉ nhận HTTP |

**3 điểm khác biệt quan trọng nhất so với Docker:**

1. **Port khai báo 1 chiều:** Docker `-p 8080:3000` map cả host lẫn container. App Service chỉ cần bạn khai báo container port (`WEBSITES_PORT`) — phía ngoài (80/443) do nó lo.

2. **Cold start là vấn đề PaaS:** Docker chạy trên server bạn kiểm soát → không bao giờ "ngủ". App Service dùng chung infrastructure → tiết kiệm tài nguyên bằng cách idle app ít traffic → sinh ra bài toán cold start.

3. **Health check từ ngoài vào:** Docker `HEALTHCHECK` chạy *bên trong* container. App Service probe HTTP endpoint *từ ngoài* → tự remove instance khỏi load balancer nếu fail.

---

## Checklist ghi nhớ cho AI-200

- [ ] Startup command override **`CMD`** trong Dockerfile, không override `ENTRYPOINT`
- [ ] Dùng `/bin/bash -c '...'` khi cần chạy nhiều lệnh tuần tự trong startup command
- [ ] App Service tự route traffic nếu container lắng nghe **port 80 hoặc 8080**
- [ ] Port khác → khai báo qua app setting **`WEBSITES_PORT`**
- [ ] App Service **xử lý TLS termination** — container chỉ nhận HTTP, không cần cấu hình SSL
- [ ] App Service chỉ expose **một port duy nhất** cho custom container
- [ ] Persistent storage **tắt mặc định** cho Linux custom container
- [ ] Bật persistent storage: `WEBSITES_ENABLE_APP_SERVICE_STORAGE=true` → dữ liệu tồn tại tại `/home`
- [ ] Khi scale out, tất cả instance **chia sẻ cùng `/home`**
- [ ] App bị idle sau ~20 phút không có traffic → cold start khi có request mới
- [ ] Always-on ngăn cold start, yêu cầu **Basic tier trở lên**
- [ ] Health check probe `/health` mỗi phút — fail 10 lần → loại instance khỏi load balancer
- [ ] Health check response: **HTTP 200** = healthy, response khác hoặc timeout = unhealthy
- [ ] Thay đổi health check config → **restart app**

---

*Bài tiếp theo: Bài 3 — Configure application settings*
