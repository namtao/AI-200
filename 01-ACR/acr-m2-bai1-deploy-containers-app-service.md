# Bài 1 — Deploy Containers to Azure App Service

> Khoá: AI-200 · ACR — Deploy containers to Azure App Service

---

## Bức tranh tổng thể

Ở module trước, bạn đã học cách lưu container image trong ACR. Module này trả lời câu hỏi tiếp theo: **làm thế nào để chạy image đó thành một web application thực sự?**

Azure App Service là một trong những cách đơn giản nhất để deploy container. Bạn cung cấp image, App Service lo phần còn lại: provisioning server, load balancing, scaling, TLS certificate, và custom domain. Bạn không cần quản lý Kubernetes hay cấu hình infrastructure phức tạp.

Về kỹ thuật, App Service chạy container của bạn trên **Linux** — đây là yêu cầu bắt buộc khi dùng Web App for Containers (không hỗ trợ Windows container).

---

## Container Image Sources — Nguồn image

Khi tạo Web App for Containers, bước đầu tiên là chỉ định **nơi App Service sẽ pull image về**. Có hai lựa chọn:

### Lựa chọn 1: Azure Container Registry (ACR) — Khuyến nghị cho production

Lý do ACR được khuyến nghị:

- **Tích hợp Microsoft Entra ID:** App Service xác thực với ACR bằng managed identity — không cần lưu password ở đâu
- **Geo-replication:** Image được lưu gần region deploy, pull nhanh hơn
- **Image scanning:** Tích hợp tính năng quét bảo mật image
- **Private network access:** Kết nối ACR qua private endpoint, không qua internet công khai

Khi chọn ACR, bạn cần chỉ định: registry nào, authentication method, image name, và tag.

### Lựa chọn 2: Other container registries

Bao gồm bất kỳ registry nào có thể truy cập qua HTTPS và hỗ trợ **Docker Registry HTTP API V2 specification** — đây là chuẩn giao tiếp giữa Docker client và registry. Các registry phổ biến đều hỗ trợ:

- **Docker Hub** — registry công khai lớn nhất, server URL: `https://index.docker.io/v1/`
- **GitHub Container Registry (GHCR)** — lưu image cùng chỗ với source code, server URL: `https://ghcr.io`
- **Self-hosted registry** — bạn tự dựng registry trên server riêng

Với public image: chỉ cần tên image.
Với private image: cần thêm server URL, username, và password.

---

## ACR Authentication — Xác thực với ACR

Đây là phần quan trọng nhất trong bài — liên quan trực tiếp đến bảo mật. App Service cần quyền pull image từ ACR, và có hai cách để cấp quyền đó:

### Cách 1: Managed Identity (Khuyến nghị cho production)

**Managed identity là gì?**

Managed identity là một "danh tính" do Azure tự quản lý. Azure tự tạo, tự xoay vòng credential, và bạn không bao giờ cần thấy hay lưu password. App Service dùng danh tính này để chứng minh với ACR rằng "tôi là App Service X, tôi có quyền pull image".

**Cơ chế hoạt động:**
1. App Service cần pull image từ ACR
2. App Service dùng managed identity để lấy token từ Microsoft Entra ID
3. Dùng token đó xác thực với ACR
4. ACR kiểm tra xem identity có role `AcrPull` không
5. Có → cho pull / Không → từ chối

Bạn không bao giờ thấy hay quản lý password — toàn bộ diễn ra tự động.

**Hai loại managed identity:**

**System-assigned managed identity:**
- Azure tự tạo identity khi bạn bật trên web app
- Identity **gắn liền với vòng đời web app** — xoá web app thì identity cũng bị xoá
- Chỉ dùng được cho web app đó
- Phù hợp khi: chỉ có một web app duy nhất cần truy cập registry

**User-assigned managed identity:**
- Bạn tự tạo identity như một Azure resource độc lập, rồi gán cho web app
- Identity **tồn tại độc lập** — xoá web app không ảnh hưởng đến identity
- Một identity có thể gán cho nhiều web app khác nhau
- Phù hợp khi: nhiều app cần cùng quyền truy cập registry, hoặc muốn cấu hình permission trước khi tạo web app

**Yêu cầu bắt buộc:** Dù dùng loại nào, managed identity đó phải được gán role **`AcrPull`** trên registry. Bước này thường do platform team thực hiện trước.

### Cách 2: Admin Credentials

ACR có một tài khoản admin với username/password. Đây là cách đơn giản hơn nhưng kém bảo mật hơn.

**Bật admin user trên ACR:**

```bash
az acr update --name myregistry --admin-enabled true
```

Sau khi bật, lấy username/password từ Portal hoặc CLI rồi điền vào cấu hình App Service.

**Nhược điểm:**
- Credential được lưu trong App Service config — ai có quyền đọc config thì thấy được
- Nếu bị lộ, phải rotate thủ công và cập nhật tất cả nơi đang dùng
- Không có audit trail rõ ràng

**Khi nào dùng:** Development và learning environment — khi cần setup nhanh, không cần bảo mật enterprise-grade.

> **Ghi nhớ cho thi:** Managed identity là **recommended cho production**. Admin credentials chỉ dùng cho development vì phải lưu password trong config.

---

## Deploy bằng Azure Portal

Portal cung cấp giao diện từng bước, phù hợp khi bạn muốn xem trực quan và xác nhận từng setting.

### Bước 1 — Basics tab

Điền các thông tin cơ bản:
- Subscription và Resource Group
- App name — trở thành subdomain: `<app-name>.azurewebsites.net`
- Region — chọn gần người dùng cuối
- **Publish: chọn "Container"** — đây là điểm khác biệt với web app thông thường
- **Operating System: chọn "Linux"** — container trên App Service chạy Linux
- App Service plan — chọn tier phù hợp (ảnh hưởng CPU, RAM, scaling)

### Bước 2 — Container tab

Đây là nơi chỉ định image source.

**Nếu dùng ACR:**
1. Image Source → chọn "Azure Container Registry"
2. Registry → chọn từ dropdown (tự động liệt kê registry cùng subscription)
3. Authentication method → "Managed Identity" hoặc "Admin credentials"
4. Image → chọn repository
5. Tag → chọn tag cụ thể (không dùng `latest` trong production)

**Nếu dùng registry khác:**
1. Image Source → chọn "Other container registries"
2. Server URL → URL của registry (vd. `https://index.docker.io/v1/` cho Docker Hub)
3. Username và Password → cho private registry (bỏ trống nếu public)
4. Full Image Name and Tag → vd. `nginx:latest` hoặc `myuser/myapp:v1`

---

## Deploy bằng Azure CLI

CLI phù hợp khi bạn muốn workflow có thể lặp lại, đưa vào script, hoặc tích hợp vào CI/CD pipeline.

### Tạo web app với image từ ACR

```bash
az webapp create \
    --resource-group myResourceGroup \
    --plan myAppServicePlan \
    --name myDocumentProcessor \
    --container-image-name myregistry.azurecr.io/docprocessor:v1
```

Giải thích từng tham số:
- `--resource-group` — resource group chứa web app
- `--plan` — App Service plan đã tồn tại (xác định tier và region)
- `--name` — tên web app, phải unique toàn cầu → thành `<name>.azurewebsites.net`
- `--container-image-name` — đường dẫn đầy đủ đến image, bao gồm tag

Lệnh này giả định App Service plan và resource group đã tồn tại, và App Service đã có quyền pull image qua managed identity.

> **Lưu ý:** Luôn chỉ định explicit tag (`:v1`) thay vì `:latest` để deployment là deterministic.

### Deploy từ Docker Hub — public image

```bash
az webapp create \
    --resource-group myResourceGroup \
    --plan myAppServicePlan \
    --name myWebApp \
    --container-image-name nginx \
    --docker-registry-server-url https://index.docker.io/v1/
```

Public image không cần username/password — chỉ cần server URL và tên image.

### Deploy từ Docker Hub — private image

```bash
az webapp create \
    --resource-group myResourceGroup \
    --plan myAppServicePlan \
    --name myWebApp \
    --container-image-name myusername/myapp:latest \
    --docker-registry-server-url https://index.docker.io/v1/ \
    --docker-registry-server-user myusername \
    --docker-registry-server-password <password>
```

Private image cần thêm `--docker-registry-server-user` và `--docker-registry-server-password`.

**GitHub Container Registry:** Thay server URL thành `https://ghcr.io`, còn lại tương tự.

---

## Deploy bằng VS Code

VS Code với Docker extension và Azure App Service extension cho phép deploy bằng cách click chuột — phù hợp trong giai đoạn phát triển khi cần iterate nhanh.

**Quy trình:**
1. Build image local bằng Docker extension
2. Push image lên registry
3. Trong tab REGISTRIES của Docker extension → chuột phải vào image tag → "Deploy Image to Azure App Service"
4. Làm theo prompt: chọn subscription, tên app, resource group, App Service plan

> **Lưu ý:** VS Code extension tiện cho development nhưng không phù hợp production. Production nên dùng CLI hoặc CI/CD pipeline để có quy trình nhất quán và có thể audit.

---

## Cập nhật Container Image

### Đổi sang version mới (tag khác)

Khi release phiên bản mới:

```bash
az webapp config container set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --container-image-name myregistry.azurecr.io/docprocessor:v2
```

App Service tự động restart và pull image mới.

### Push image mới vào cùng tag

**App Service KHÔNG tự động phát hiện** khi bạn push image mới vào cùng tag (ví dụ: đè lên `latest`).

App Service chỉ chạy `docker pull` khi restart, và ngay cả khi đó chỉ pull các **layer đã thay đổi** — không pull lại toàn bộ nếu tag không đổi.

Để App Service nhận image mới trong trường hợp này:
1. **Restart thủ công** web app, hoặc
2. **Bật continuous deployment** để webhook tự động trigger restart

---

## Continuous Deployment — Tự động deploy khi push image mới

Continuous deployment cho phép App Service tự động pull và deploy image mới mỗi khi bạn push lên registry.

### Bước 1 — Bật continuous deployment

```bash
az webapp deployment container config \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --enable-cd true
```

Lệnh này trả về một **webhook URL**.

### Bước 2 — Cấu hình webhook trong ACR

```bash
az acr webhook create \
    --registry myregistry \
    --name appservicehook \
    --uri <webhook-url-từ-bước-trên> \
    --actions push
```

**Luồng hoạt động:**
```
Developer push image mới lên ACR
    → ACR kích hoạt webhook
    → App Service nhận thông báo
    → App Service restart và pull image mới
    → Web app chạy với image mới
```

> **Continuous deployment nên dùng ở đâu?**
> Staging/dev: rất tiện, mọi build mới được test tự động.
> Production: cần cân nhắc — thường muốn kiểm soát rõ ràng khi nào deploy lên production. Nếu dùng, nên kết hợp với approval gate trong CI/CD pipeline.

---

## Image Pull Behavior — Khi nào App Service pull image?

Hiểu cơ chế này giúp bạn dự đoán behavior khi deploy và troubleshoot khi có vấn đề.

| Tình huống | App Service làm gì |
|---|---|
| **Initial deployment** — lần đầu deploy hoặc đổi image reference | Pull toàn bộ image layers từ registry |
| **App restart** — restart thủ công hoặc do crash | Chạy `docker pull`, chỉ pull layers đã thay đổi. Nếu image không đổi → dùng cached layers |
| **Scale out** — thêm instance mới | Instance mới pull image. Có thể pull full image nếu layers chưa được cache trên infrastructure đó |
| **Pricing tier change** — đổi App Service plan tier | Azure có thể cấp infrastructure mới → pull image từ đầu, ảnh hưởng startup time |

**Ý nghĩa thực tế:**
- Khi scale out, instance mới có thể mất nhiều thời gian khởi động hơn nếu phải pull full image lần đầu
- Thay đổi pricing tier nên thực hiện trong maintenance window vì có thể ảnh hưởng startup time

---

## Verify Deployment — Kiểm tra sau khi deploy

Lấy URL của web app:

```bash
az webapp show \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --query defaultHostName \
    --output tsv
```

Trả về hostname dạng `myDocumentProcessor.azurewebsites.net`. Mở URL trong browser hoặc dùng `curl` kiểm tra app có respond không.

Nếu container không start được, xem log:

```bash
az webapp log tail \
    --resource-group myResourceGroup \
    --name myDocumentProcessor
```

---

## Tóm tắt luồng deploy hoàn chỉnh

```
1. Build image           → az acr build (module ACR bài 2)
2. Image lưu trong ACR   → myregistry.azurecr.io/docprocessor:v1
3. Tạo App Service plan  → az appservice plan create
4. Deploy web app        → az webapp create --container-image-name ...
5. App Service pull image → tự động khi tạo web app
6. Web app chạy          → https://myDocumentProcessor.azurewebsites.net
7. Release version mới   → az webapp config container set ... :v2
8. Bật auto-deploy       → az webapp deployment container config --enable-cd true
```

---

## Bản chất bài này là gì?

**Một câu:** App Service là "Docker host được quản lý" — bạn cung cấp image, Azure lo toàn bộ phần còn lại.

Nếu với Docker thuần bạn phải `docker run` trên một server nào đó, thì App Service *là* server đó — chỉ có điều bạn không cần nhìn thấy nó. Bạn chỉ khai báo "tôi muốn chạy image này", Azure tự lo provisioning, load balancing, TLS, scaling.

### So sánh với Docker thuần

| Khía cạnh | Docker thuần | Azure App Service |
|---|---|---|
| Chạy container | `docker run` trên máy/server tự quản | Azure tự quản lý host |
| Truy cập từ internet | Tự config reverse proxy, port, TLS | Tự động có HTTPS + custom domain |
| Scaling | Tự cấu hình (compose, swarm, k8s) | Tự động theo App Service plan |
| Xác thực với registry | `docker login` — phải lưu credential | Managed identity — không cần credential |
| Cập nhật image mới | SSH vào server, pull + restart | `az webapp config container set` hoặc webhook |
| Infrastructure | Bạn phải có host | Azure lo hoàn toàn |

**Mental model:** Docker = toolset để build/run container. App Service = platform PaaS nhận image của bạn và chạy nó như một web app production-ready.

**Điểm khác biệt cốt lõi so với Docker:**
- `docker login` → Managed Identity (không bao giờ thấy password)
- `docker pull + docker run` trên server → App Service plan tự lo
- SSH vào server để cập nhật → webhook từ registry tự trigger restart
- Port mapping thủ công → App Service expose port 80/443 tự động (container lắng nghe port gì thì App Service biết qua biến môi trường `WEBSITES_PORT`)

---

## Checklist ghi nhớ cho AI-200

- [ ] App Service hỗ trợ hai nguồn image: **ACR** (recommended) và **Other registries** (Docker Hub, GHCR, self-hosted)
- [ ] Registry khác đều được miễn là hỗ trợ **Docker Registry HTTP API V2**
- [ ] Container trên App Service chạy trên **Linux**
- [ ] ACR authentication: **Managed Identity** (production) vs **Admin credentials** (dev)
- [ ] **System-assigned identity** gắn với vòng đời web app · **User-assigned identity** tồn tại độc lập, dùng được cho nhiều app
- [ ] Managed identity cần được gán role **`AcrPull`** trên registry
- [ ] Bật admin credentials: `az acr update --name myregistry --admin-enabled true`
- [ ] Tạo web app với image: `az webapp create --container-image-name <registry>/<image>:<tag>`
- [ ] Cập nhật image version mới: `az webapp config container set --container-image-name ...<new-tag>`
- [ ] App Service **KHÔNG tự detect** khi push image mới vào cùng tag — phải restart thủ công hoặc dùng webhook
- [ ] Bật continuous deployment: `az webapp deployment container config --enable-cd true` → nhận webhook URL
- [ ] Webhook URL từ App Service cần được cấu hình trong ACR để trigger auto-deploy
- [ ] Khi scale out, instance mới có thể cần pull full image → ảnh hưởng startup time
- [ ] `az webapp show --query defaultHostName` để lấy URL web app sau khi deploy

---

*Bài tiếp theo: Bài 2 — Configure container runtime behavior*
