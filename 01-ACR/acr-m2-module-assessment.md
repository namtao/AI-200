# Module Assessment — Deploy Containers to Azure App Service

> Khoá: AI-200 · ACR — Deploy containers to Azure App Service

---

## Câu 1

**A container image is configured to listen on port 8000. After deploying to App Service, requests to the application return connection errors. What configuration change resolves this issue?**

- Set the `WEBSITES_PORT` app setting to 8000. ✅
- Modify the Dockerfile to use EXPOSE 80.
- Enable the HTTP/2 protocol in the platform settings.

**Đáp án: Set the `WEBSITES_PORT` app setting to 8000**

App Service mặc định tự route traffic vào port 80 hoặc 8080. Nếu container lắng nghe port khác (ở đây là 8000), App Service không biết để forward request đến đúng nơi → connection error.

```bash
az webapp config appsettings set \
    --resource-group myResourceGroup \
    --name myApp \
    --settings WEBSITES_PORT=8000
```

Tại sao các đáp án còn lại sai:
- **Modify Dockerfile to EXPOSE 80:** `EXPOSE` trong Dockerfile chỉ là metadata documentation — nó không thực sự thay đổi port app lắng nghe. App vẫn chạy trên 8000, chỉ Dockerfile "nói" là 80. Hơn nữa, sửa Dockerfile đòi hỏi rebuild và redeploy image — không cần thiết khi chỉ cần set app setting.
- **Enable HTTP/2:** HTTP/2 là về protocol version, không liên quan gì đến port mismatch.

---

## Câu 2

**A document processing application writes output files during processing. After a container restart, the output files are missing. How do you configure App Service to persist these files?**

- Set `WEBSITES_ENABLE_APP_SERVICE_STORAGE` to true and write files to the `/home` directory. ✅
- Configure a larger container image with more disk space.
- Enable always-on to prevent container restarts.

**Đáp án: Set `WEBSITES_ENABLE_APP_SERVICE_STORAGE` to true và ghi file vào `/home`**

Mặc định, file system của container là **ephemeral** — mọi file ghi vào bị mất khi restart. `WEBSITES_ENABLE_APP_SERVICE_STORAGE=true` mount persistent storage tại `/home`, dữ liệu tại đây tồn tại qua restart.

```bash
az webapp config appsettings set \
    --resource-group myResourceGroup \
    --name myApp \
    --settings WEBSITES_ENABLE_APP_SERVICE_STORAGE=true
```

App cần ghi file vào `/home/output/` (hoặc bất kỳ subdirectory nào trong `/home`).

Tại sao các đáp án còn lại sai:
- **Larger container image with more disk space:** Kích thước image không giải quyết được vấn đề ephemeral — dù image to đến đâu, file ghi lúc runtime vẫn mất khi restart vì đây là đặc tính của container layer, không phải dung lượng.
- **Enable always-on:** Always-on giúp app không bị idle và giảm cold start, nhưng **không ngăn được restart** — app vẫn có thể restart khi crash, deploy mới, hay Azure maintenance. Giải pháp đúng phải là persistent storage, không phải tránh restart.

---

## Câu 3

**A production application requires different API endpoint URLs for staging and production deployment slots. Which configuration approach ensures the staging URL doesn't swap to production during a slot swap?**

- Configure the API_ENDPOINT setting as a slot setting. ✅
- Store the API endpoint in the container image for each environment.
- Use connection strings instead of app settings for the API endpoint.

**Đáp án: Configure the API_ENDPOINT setting as a slot setting**

Khi swap hai slot, mặc định **tất cả app settings đều swap** cùng với code. Điều này có nghĩa staging API URL sẽ vào production và ngược lại — không mong muốn.

Slot setting **gắn với slot, không swap**:

```bash
# Cấu hình cho staging slot
az webapp config appsettings set \
    --resource-group myResourceGroup \
    --name myApp \
    --slot staging \
    --slot-settings API_ENDPOINT=https://api-staging.example.com

# Cấu hình cho production slot
az webapp config appsettings set \
    --resource-group myResourceGroup \
    --name myApp \
    --slot-settings API_ENDPOINT=https://api.example.com
```

Sau swap: mỗi slot vẫn giữ API_ENDPOINT của mình.

Tại sao các đáp án còn lại sai:
- **Store in container image:** Bake config vào image vi phạm nguyên tắc "same image for all environments" — phải build riêng image cho staging và production, mất toàn bộ lợi ích của containerization. Nếu API URL thay đổi, phải rebuild image.
- **Use connection strings:** Connection strings cũng swap theo khi swap slot, trừ khi cũng được đánh dấu là slot setting. Việc đổi từ app settings sang connection strings không giải quyết được vấn đề swap — và connection strings vốn thiết kế cho database, không phải API endpoint.

---

## Câu 4

**A container starts successfully but health checks are failing, and App Service removes instances from the load balancer. The application has a `/healthz` endpoint that returns HTTP 200. What is the most likely cause?**

- The health check path in App Service is configured differently than the application's health endpoint path. ✅
- Health checks require HTTPS, but the container only serves HTTP.
- The container needs more memory to handle health check requests.

**Đáp án: Health check path được cấu hình khác với endpoint thực tế của app**

App có endpoint `/healthz` nhưng App Service đang probe một path khác — ví dụ `/health` (default) hoặc một path nào đó bị cấu hình nhầm. App Service nhận 404 thay vì 200 → coi là unhealthy → loại khỏi load balancer.

Giải pháp: đảm bảo health check path trong App Service khớp chính xác với endpoint trong app:

```bash
az webapp config set \
    --resource-group myResourceGroup \
    --name myApp \
    --generic-configurations '{"healthCheckPath": "/healthz"}'
```

Tại sao các đáp án còn lại sai:
- **Health checks require HTTPS:** App Service thực hiện health check từ **bên trong platform** — nó gọi trực tiếp vào container qua HTTP (không qua internet public). App Service đã xử lý TLS termination, container chỉ cần serve HTTP. Health check không yêu cầu HTTPS từ phía container.
- **Container needs more memory:** Nếu thiếu memory, container sẽ crash hoàn toàn và logs sẽ hiện OOM (Out of Memory) error. Scenario này mô tả container start thành công và endpoint hoạt động — vấn đề là path mismatch, không phải resource.

---

## Câu 5

**A developer needs to verify that app settings are correctly injected into a running container. Which diagnostic tool provides this information?**

- The Kudu diagnostic console Environment page. ✅
- The log stream in the Azure portal.
- The App Service metrics dashboard.

**Đáp án: The Kudu diagnostic console Environment page**

Kudu/SCM site có trang Environment hiển thị **toàn bộ environment variable** mà App Service inject vào container — bao gồm app settings, connection strings, Key Vault resolved values, và system variables.

Truy cập: `https://<app-name>.scm.azurewebsites.net/Env`

Đây là cách nhanh nhất để xác nhận:
- Setting đã được save đúng chưa
- Key Vault reference đã được resolve thành giá trị thực chưa
- Có typo trong tên setting không
- Giá trị có đúng với kỳ vọng không

Tại sao các đáp án còn lại sai:
- **Log stream:** Log stream hiển thị output từ stdout/stderr của container — tức là những gì app *ghi ra*, không phải environment variable được inject vào. Trừ khi app chủ động log ra toàn bộ env var lúc startup, log stream không giúp được ở đây.
- **App Service metrics dashboard:** Metrics dashboard hiển thị performance metrics: CPU, memory, request count, response time... Không có thông tin về configuration hay environment variable.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Port configuration — container lắng nghe port khác 80/8080 | `WEBSITES_PORT=8000` |
| 2 | Persistent storage — file mất sau restart | `WEBSITES_ENABLE_APP_SERVICE_STORAGE=true` + ghi vào `/home` |
| 3 | Slot settings — config không swap khi swap slot | Đánh dấu `API_ENDPOINT` là slot setting |
| 4 | Health check — path mismatch | Sửa health check path khớp với `/healthz` |
| 5 | Diagnostic — verify env var được inject đúng | Kudu Environment page |

---

*Module hoàn thành. Learning Path tiếp theo: Deploy and manage apps on Azure Container Apps (ACA)*
