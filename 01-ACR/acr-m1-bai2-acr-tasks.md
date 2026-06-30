# Bài 2 — Build and Run Images với ACR Tasks

> Khoá: AI-200 · Module: Implement container application hosting on Azure

---

## Vấn đề ACR Tasks giải quyết

Khi làm việc nhóm với container, bạn thường gặp các vấn đề sau:

- Developer A build image trên macOS, developer B build trên Windows → kết quả có thể khác nhau
- Mỗi người phải cài Docker Desktop trên máy → tốn tài nguyên, phức tạp khi onboard người mới
- Muốn tự động build mỗi khi có commit lên GitHub nhưng phải tự thiết lập CI/CD riêng
- Base image (ví dụ `python:3.11`) có bản vá bảo mật mới → phải tự nhớ rebuild toàn bộ image phụ thuộc

**ACR Tasks giải quyết tất cả những điều trên** bằng cách chuyển toàn bộ quá trình build lên Azure cloud.

---

## ACR Tasks là gì?

ACR Tasks là một bộ tính năng tích hợp sẵn trong Azure Container Registry, cho phép bạn **build container image trực tiếp trên Azure** thay vì build trên máy local.

**Luồng truyền thống (không dùng ACR Tasks):**
```
Máy local → docker build → docker push → ACR
```

**Luồng với ACR Tasks:**
```
Máy local / GitHub → az acr build (upload source) → Azure build → ACR
```

Bạn chỉ cần gửi source code lên, Azure lo phần còn lại.

### Lợi ích cụ thể

**1. Consistent builds — build nhất quán**

Mọi build đều chạy trên cùng một infrastructure của Azure, bất kể máy tính của developer là gì. Không còn tình trạng "chạy được trên máy tôi nhưng không chạy được trên server".

**2. Không cần cài Docker local**

Developer không cần cài Docker Desktop. Điều này đặc biệt hữu ích khi:
- Onboard thành viên mới vào team
- Build từ môi trường CI/CD không có Docker
- Làm việc trên máy tính bị hạn chế quyền cài phần mềm

**3. CI/CD tích hợp sẵn**

ACR Tasks tích hợp trực tiếp với GitHub và Azure DevOps. Bạn có thể cấu hình để mỗi khi có commit lên nhánh `main`, Azure tự động build image mới mà không cần thêm bước cấu hình CI/CD riêng.

**4. Multi-platform support**

ACR Tasks có thể build image cho nhiều kiến trúc: Linux, Windows, và cả ARM64 (dùng cho edge AI deployment trên thiết bị như Raspberry Pi, Jetson Nano). Bạn không cần duy trì nhiều môi trường build khác nhau.

---

## 3 loại ACR Tasks

ACR Tasks chia thành 3 loại chính, phù hợp với các tình huống khác nhau:

```
ACR Tasks
├── 1. Quick Tasks         → build ngay lập tức, một lần, không cần setup
├── 2. Triggered Tasks     → tự động chạy khi có sự kiện xảy ra
│   ├── Source code trigger    (commit hoặc pull request lên Git)
│   ├── Base image trigger     (base image trong Dockerfile được cập nhật)
│   └── Scheduled trigger      (chạy theo lịch cố định, dùng cron)
└── 3. Multi-step Tasks    → workflow nhiều bước: build → test → push
```

---

## Loại 1: Quick Tasks — build on-demand

Quick task là cách nhanh nhất để build và push một image. Bạn chỉ cần một lệnh duy nhất, không cần tạo task hay cấu hình gì trước.

### Lệnh cơ bản

```bash
az acr build --registry myregistry --image inference-api:v1.0.0 .
```

Giải thích từng phần:
- `az acr build` — lệnh Azure CLI để build image qua ACR Tasks
- `--registry myregistry` — tên registry ACR của bạn
- `--image inference-api:v1.0.0` — tên và tag của image sẽ được tạo ra
- `.` — build context, dấu chấm nghĩa là "dùng thư mục hiện tại"

Khi chạy lệnh này, ACR sẽ:
1. Nén toàn bộ thư mục hiện tại thành một archive
2. Upload archive đó lên Azure
3. Chạy `docker build` trên cloud theo Dockerfile trong thư mục
4. Push image vào registry với tag `inference-api:v1.0.0`

### Build context là gì?

Build context là **tập hợp file mà ACR sẽ dùng để build image**. Đây là khái niệm quan trọng vì nó quyết định Azure sẽ nhận được những file nào.

ACR Tasks chấp nhận build context từ 3 nguồn:

**a) Local directory — thư mục trên máy bạn**

```bash
az acr build --registry myregistry --image inference-api:v1.0.0 .
```

Dấu `.` là thư mục hiện tại. Bạn cũng có thể chỉ định đường dẫn cụ thể:

```bash
az acr build --registry myregistry --image inference-api:v1.0.0 ./my-app/
```

ACR sẽ nén toàn bộ thư mục đó và upload lên. Vì vậy nên dùng `.dockerignore` để loại bỏ các file không cần thiết (xem phần Best Practices).

**b) Git repository — repo GitHub hoặc Azure DevOps**

```bash
az acr build --registry myregistry \
  --image inference-api:v1.0.0 \
  https://github.com/myorg/inference-api.git
```

Khi dùng Git URL, ACR sẽ clone repo đó về phía Azure rồi build. Bạn không cần clone về máy local. Điều này rất tiện khi:
- Làm việc từ xa và không muốn clone repo lớn
- Chạy build từ môi trường CI/CD
- Repo quá lớn để upload từ máy local

Bạn cũng có thể chỉ định nhánh cụ thể bằng cách thêm `#tên-nhánh` vào cuối URL:

```bash
az acr build --registry myregistry \
  --image inference-api:v1.0.0 \
  https://github.com/myorg/inference-api.git#feature/new-model
```

**c) Remote tarball — file nén trên web server**

```bash
az acr build --registry myregistry \
  --image inference-api:v1.0.0 \
  https://example.com/source.tar.gz
```

Dùng khi source code được đóng gói sẵn thành file `.tar.gz` và lưu trên một URL có thể truy cập công khai.

### Khi nào nên dùng Quick Tasks?

Quick tasks phù hợp nhất khi bạn cần build **ngay lập tức, không lặp lại**:

- Đang sửa Dockerfile và muốn kiểm tra xem build có thành công không trước khi commit
- Cần build một image thử nghiệm nhanh
- Chạy build một lần trong pipeline CI/CD đơn giản
- Thử nghiệm base image mới hoặc dependency mới

Quick tasks **không phù hợp** khi bạn cần build tự động liên tục — hãy dùng Triggered Tasks cho trường hợp đó.

---

## Loại 2: Automatically Triggered Tasks — tự động theo sự kiện

Triggered tasks cho phép ACR **tự động chạy build mà không cần bạn ra lệnh thủ công**. Có 3 loại trigger:

---

### 2a. Source code trigger — trigger từ Git commit

Source code trigger tự động build image mỗi khi có **commit mới hoặc pull request** được tạo trong GitHub hoặc Azure DevOps.

Cách hoạt động:
1. Bạn tạo một ACR Task và liên kết với Git repo
2. ACR tự động tạo một webhook trong repo đó
3. Mỗi khi có commit, GitHub gửi thông báo đến ACR qua webhook
4. ACR tự động chạy build

**Lệnh tạo task với source code trigger:**

```bash
az acr task create \
  --registry myregistry \
  --name build-inference-api \
  --image inference-api:{{.Run.ID}} \
  --context https://github.com/myorg/inference-api.git#main \
  --file Dockerfile \
  --git-access-token $PAT
```

Giải thích các tham số:
- `--name build-inference-api` — tên của task, dùng để quản lý sau này
- `--image inference-api:{{.Run.ID}}` — `{{.Run.ID}}` là biến đặc biệt, mỗi lần build sẽ tạo ra một ID duy nhất làm tag (ví dụ: `inference-api:cb1`). Cách này giúp bạn trace được mỗi image được build từ lần chạy nào
- `--context .../inference-api.git#main` — URL repo + `#main` chỉ định theo dõi nhánh `main`
- `--file Dockerfile` — tên file Dockerfile trong repo (mặc định là `Dockerfile` nếu bỏ qua)
- `--git-access-token $PAT` — Personal Access Token để ACR có quyền đọc repo và tạo webhook

> **Personal Access Token (PAT) là gì?**
> PAT là một chuỗi token bạn tạo ra từ GitHub/Azure DevOps để cấp quyền cho ứng dụng bên thứ ba (ở đây là ACR) truy cập repo của bạn. Nó thay thế cho username/password. ACR cần PAT để clone repo và tạo webhook.

> ⚠️ **Bảo mật PAT:** Không bao giờ hardcode PAT vào script hay commit lên repo. Luôn lưu trong **Azure Key Vault** và tham chiếu từ đó.

---

### 2b. Base image trigger — trigger khi base image thay đổi

Đây là tính năng đặc biệt quan trọng với AI/ML application.

**Vấn đề:** Dockerfile của bạn thường bắt đầu bằng một base image:

```dockerfile
FROM python:3.11
# hoặc
FROM pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime
```

Khi nhà cung cấp phát hành bản vá bảo mật cho `python:3.11`, nếu bạn không rebuild image của mình thì bạn vẫn đang dùng base image cũ có lỗ hổng bảo mật.

**Giải pháp:** Base image trigger tự động phát hiện khi base image được cập nhật và **tự động rebuild toàn bộ image** có `FROM <base-image>` trong Dockerfile.

Cách hoạt động:
1. Bạn tạo ACR Task cho application image (ví dụ `inference-api`)
2. ACR theo dõi base image trong câu lệnh `FROM` của Dockerfile
3. Khi `python:3.11` được push phiên bản mới lên Docker Hub (hoặc ACR), ACR phát hiện thay đổi
4. ACR tự động trigger build lại `inference-api`

Không cần lệnh riêng để bật base image trigger — nó **tự động được kích hoạt** khi bạn tạo task bằng `az acr task create`.

> **Lưu ý quan trọng về phạm vi hoạt động:**
> - Base image trong **cùng ACR registry**: trigger hoạt động tốt nhất, phát hiện gần như ngay lập tức
> - Base image từ **Docker Hub** hoặc public registry khác: ACR vẫn có thể detect nhưng có thể chậm hơn
> - Base image từ **private registry khác**: cần cấu hình thêm

---

### 2c. Scheduled trigger — trigger theo lịch

Scheduled trigger cho phép ACR Task chạy theo lịch cố định, dùng **cron expression** — một cú pháp chuẩn để mô tả lịch thời gian.

**Cú pháp cron expression:**

```
┌─────── phút (0-59)
│  ┌──── giờ (0-23, UTC)
│  │  ┌─ ngày trong tháng (1-31)
│  │  │  ┌ tháng (1-12)
│  │  │  │  ┌ ngày trong tuần (0-7, 0 và 7 đều là Chủ nhật)
│  │  │  │  │
*  *  *  *  *
```

Một số ví dụ thông dụng:

| Cron expression | Ý nghĩa |
|---|---|
| `0 0 * * *` | Hàng ngày lúc 00:00 UTC (nightly) |
| `0 2 * * 1` | Mỗi thứ Hai lúc 02:00 UTC |
| `0 */6 * * *` | Mỗi 6 giờ một lần |
| `30 8 * * 1-5` | 08:30 UTC các ngày trong tuần |

**Lệnh tạo scheduled task:**

```bash
az acr task create \
  --registry myregistry \
  --name nightly-build \
  --image inference-api:nightly \
  --context https://github.com/myorg/inference-api.git \
  --schedule "0 0 * * *" \
  --file Dockerfile \
  --git-access-token $PAT
```

Lệnh này tạo một task tên `nightly-build`, mỗi ngày lúc nửa đêm UTC sẽ build lại image `inference-api:nightly` từ nhánh mặc định của repo.

**Khi nào dùng scheduled trigger:**

- **Nightly build:** Build mỗi tối để lấy các dependency cập nhật mới nhất trong ngày
- **Patch pickup định kỳ:** Dù base image trigger hoạt động, đặt thêm scheduled trigger như "lưới an toàn" để đảm bảo base image patch luôn được áp dụng
- **Security scan:** Chạy công cụ quét bảo mật image định kỳ
- **Cleanup:** Xoá untagged manifest cũ để kiểm soát chi phí storage

---

## Loại 3: Multi-step Tasks — workflow nhiều bước

Multi-step tasks mở rộng khả năng của ACR Tasks bằng cách cho phép định nghĩa một **workflow tuần tự** gồm nhiều bước trong một file YAML.

Mỗi bước có thể:
- **Build** một image
- **Push** image lên registry
- **Run** một container để thực thi lệnh (ví dụ: chạy test)

### Ví dụ YAML file — build, push, rồi chạy test

```yaml
version: v1.1.0
steps:
  - build: -t {{.Run.Registry}}/inference-api:{{.Run.ID}} .

  - push:
    - {{.Run.Registry}}/inference-api:{{.Run.ID}}

  - cmd: {{.Run.Registry}}/inference-api:{{.Run.ID}} python -m pytest tests/
```

Giải thích từng bước:

**Step 1 — build:**
```yaml
- build: -t {{.Run.Registry}}/inference-api:{{.Run.ID}} .
```
Build image từ Dockerfile trong thư mục hiện tại. `{{.Run.Registry}}` là biến tự động trỏ đến registry của bạn, `{{.Run.ID}}` là ID duy nhất của lần chạy này.

**Step 2 — push:**
```yaml
- push:
  - {{.Run.Registry}}/inference-api:{{.Run.ID}}
```
Push image vừa build lên registry. Bước này chỉ chạy nếu build thành công.

**Step 3 — cmd (run container):**
```yaml
- cmd: {{.Run.Registry}}/inference-api:{{.Run.ID}} python -m pytest tests/
```
Chạy container từ image vừa push, thực thi lệnh `python -m pytest tests/` bên trong container. Nếu pytest fail thì ACR biết image này có vấn đề trước khi dùng vào deployment.

### Tại sao cần multi-step?

Với Docker thông thường, sau khi build xong bạn không có cơ chế tích hợp để chạy test trước khi push. Với multi-step tasks, toàn bộ pipeline **build → validate → push** xảy ra trong ACR, không cần thiết lập CI/CD riêng.

### Tính năng nâng cao của multi-step

- **Sequential steps có dependency:** Bước sau chỉ chạy khi bước trước thành công
- **Parallel execution:** Các bước độc lập có thể chạy song song để tiết kiệm thời gian
- **Conditional logic:** Chạy bước tiếp theo hay không tuỳ thuộc vào kết quả bước trước
- **Nhiều image trong một workflow:** Build nhiều image khác nhau trong cùng một pipeline

### Lệnh tạo multi-step task

```bash
az acr task create \
  --registry myregistry \
  --name build-test-push \
  --context https://github.com/myorg/inference-api.git \
  --file acr-task.yaml \
  --git-access-token $PAT
```

Khác với quick task và triggered task, bạn truyền file YAML vào `--file` thay vì tên Dockerfile.

---

## Chạy container để kiểm tra image

Ngoài việc build, ACR Tasks cũng cho phép bạn **chạy thử một container** từ image có sẵn trong registry để kiểm tra trước khi deploy.

```bash
az acr run --registry myregistry \
  --cmd 'inference-api:v1.0.0 python --version' \
  /dev/null
```

Giải thích:
- `az acr run` — lệnh để chạy container trong ACR Tasks
- `--cmd 'inference-api:v1.0.0 python --version'` — chạy lệnh `python --version` bên trong container của image `inference-api:v1.0.0`
- `/dev/null` — nghĩa là "không cần upload file nào cả", vì bạn chỉ chạy image đã có sẵn trong registry, không cần source code

**Dùng `az acr run` để:**
- Kiểm tra image có khởi động được không
- Xác nhận Python/framework/CUDA đúng version
- Chạy smoke test nhanh
- Thực thi health check command

---

## Run Variables — biến đặc biệt trong ACR Tasks

Trong các lệnh và YAML của ACR Tasks, bạn có thể dùng các biến tự động sau:

| Biến | Ý nghĩa | Ví dụ giá trị |
|---|---|---|
| `{{.Run.ID}}` | ID duy nhất của lần chạy task | `cb1`, `cb2`, ... |
| `{{.Run.Date}}` | Ngày giờ chạy task | `20240115-143022` |
| `{{.Run.Registry}}` | Tên đầy đủ của registry | `myregistry.azurecr.io` |
| `{{.Run.RegistryName}}` | Chỉ tên registry (không có `.azurecr.io`) | `myregistry` |

Việc dùng `{{.Run.ID}}` làm tag image rất quan trọng vì nó tạo ra **traceability** — bạn có thể biết image nào được build từ task run nào.

---

## Best Practices

### 1. Dùng run variables cho tag

```bash
# Thay vì dùng tag cố định:
--image inference-api:latest

# Dùng run ID để traceable:
--image inference-api:{{.Run.ID}}
```

Tag cố định như `latest` bị ghi đè mỗi lần build, khó trace. Tag dùng `{{.Run.ID}}` tạo ra lịch sử rõ ràng.

### 2. Bảo mật Personal Access Token

PAT là credential nhạy cảm. Luôn lưu trong **Azure Key Vault**:

```bash
# Sai — hardcode trong script
--git-access-token ghp_abc123xyz...

# Đúng — tham chiếu từ Key Vault
--git-access-token $(az keyvault secret show --vault-name myvault --name git-pat --query value -o tsv)
```

Không bao giờ commit PAT lên Git repository.

### 3. Theo dõi log build

Khi build fail, xem log để debug:

```bash
az acr task logs --registry myregistry --name build-inference-api
```

Log hiển thị toàn bộ quá trình build, bao gồm lỗi cụ thể ở bước nào.

### 4. Tối ưu build context với .dockerignore

Khi dùng local directory làm context, ACR upload toàn bộ thư mục. Nếu thư mục có nhiều file không cần thiết (model weights, dataset, file log...) thì upload sẽ rất chậm.

Tạo file `.dockerignore` để loại trừ:

```
# .dockerignore
__pycache__/
*.pyc
.git/
data/
*.log
node_modules/
*.pt        # PyTorch model weights — thường rất lớn
*.h5        # Keras model weights
```

### 5. Tối ưu layer caching trong Dockerfile

ACR Tasks cache các layer giữa các lần build. Để tận dụng cache tối đa, **đặt instruction ít thay đổi lên đầu Dockerfile**:

```dockerfile
# Tốt — dependency ít thay đổi đặt trước
FROM python:3.11
COPY requirements.txt .          # ít thay đổi
RUN pip install -r requirements.txt  # ít thay đổi
COPY . .                         # thường xuyên thay đổi — đặt cuối
```

```dockerfile
# Xấu — copy source trước thì cache bị invalidate mỗi lần sửa code
FROM python:3.11
COPY . .                         # thường xuyên thay đổi
RUN pip install -r requirements.txt  # bị chạy lại mỗi lần dù requirements chưa đổi
```

---

## So sánh 3 loại ACR Tasks

| | Quick Task | Triggered Task | Multi-step Task |
|---|---|---|---|
| Cách kích hoạt | Thủ công, một lần | Tự động theo sự kiện | Thủ công hoặc tự động |
| Cấu hình | Chỉ cần 1 lệnh CLI | Tạo task với `az acr task create` | File YAML + `az acr task create` |
| Phù hợp với | Dev, thử nghiệm | CI/CD automation | Build + test + push pipeline |
| Có thể trigger tự động | ❌ | ✅ | ✅ |
| Nhiều bước tuần tự | ❌ | ❌ | ✅ |
| Cần setup trước | ❌ | ✅ | ✅ |

---

## So sánh với Docker thuần

| Tính năng | Docker local | ACR Tasks |
|---|---|---|
| Cần Docker cài trên máy | ✅ bắt buộc | ❌ không cần |
| Build image | `docker build` | `az acr build` |
| Push image | `docker push` (riêng biệt) | Tự động sau build |
| Tự build khi có commit | ❌ cần CI/CD riêng | ✅ Source code trigger |
| Tự rebuild khi base image đổi | ❌ phải tự nhớ | ✅ Base image trigger |
| Build theo lịch | ❌ | ✅ Scheduled trigger |
| Build → Test → Push trong 1 workflow | ❌ | ✅ Multi-step Tasks |
| Build ARM64 trên máy x86 | ⚠️ phức tạp | ✅ hỗ trợ sẵn |

---

## Bản chất của Cloud Build — ACR Tasks hay GitHub Actions đều là một thứ

Sau khi hiểu ACR Tasks, một câu hỏi tự nhiên xuất hiện: *GitHub Actions cũng build được image, cần ACR Tasks làm gì?*

Câu trả lời ngắn: **bản chất giống nhau — đều là chạy `docker build` trên một máy Linux trên cloud thay vì máy local của bạn.**

### Vấn đề gốc rễ mà cả hai giải quyết

Khi bạn chạy `docker build` trên máy local:

```
Máy local (macOS/Windows/Linux) → docker build → image
```

Kết quả phụ thuộc vào môi trường máy bạn, version Docker, kiến trúc CPU. Developer A và B có thể ra image khác nhau từ cùng một source code.

Khi bạn chuyển build lên cloud — dù là ACR Tasks hay GitHub Actions — thực chất là:

```
Code của bạn → upload lên → Linux VM trên cloud → docker build → image
```

Máy Linux đó luôn là cùng một loại môi trường. Kết quả nhất quán cho mọi người trong team.

### So sánh bản chất giữa các tool

| Công đoạn | Máy local | GitHub Actions | ACR Tasks |
|---|---|---|---|
| Môi trường build | Máy của từng dev (macOS/Win/Linux) | Linux VM do GitHub cung cấp | Linux VM do Azure cung cấp |
| Cần cài Docker | ✅ bắt buộc | ❌ runner đã có sẵn | ❌ không cần |
| Ai lo VM | Bạn tự lo | GitHub lo | Azure lo |
| Trigger tự động | ❌ | ✅ push/PR/schedule | ✅ push/base image update/schedule |
| Tích hợp với registry | Manual push | Push thủ công hoặc thêm step | Push tự động vào ACR |
| Phù hợp khi | Đang phát triển, thử nghiệm | Dùng GitHub, không cần Azure | Đã dùng Azure ecosystem |


### Vậy khi nào nên dùng cái nào?

**Dùng GitHub Actions thuần** khi:
- Team dùng GitHub
- Không cần tích hợp sâu với Azure
- Muốn một giải pháp đơn giản, phổ biến, nhiều tài liệu

**Dùng ACR Tasks** khi:
- Team đã dùng Azure (AKS, App Service, Azure DevOps)
- Muốn base image trigger — tự động rebuild khi base image có security patch
- Muốn tất cả build, registry, và deploy nằm trong một ecosystem Azure
- Không muốn viết và maintain GitHub Actions workflow

**Điểm mấu chốt:** Không có gì magic trong ACR Tasks. Nó là một managed build service, giống GitHub Actions nhưng baked vào Azure. Nếu bạn đã hiểu GitHub Actions, bạn đã hiểu 80% cách ACR Tasks hoạt động.

---

## Checklist ghi nhớ cho AI-200

- [ ] ACR Tasks cho phép build image trên Azure mà **không cần Docker cài local**
- [ ] `az acr build` = quick task, build on-demand một lần
- [ ] Build context có thể là: local directory, Git repo URL, hoặc remote tarball URL
- [ ] `{{.Run.ID}}` tạo tag duy nhất mỗi lần build — dùng thay cho `latest` trong production
- [ ] **Source code trigger** = tự động build khi có Git commit/PR
- [ ] **Base image trigger** = tự động rebuild khi base image (trong `FROM`) được cập nhật — quan trọng với AI/ML để nhận security patch
- [ ] Base image trigger hoạt động tốt nhất khi cả hai image **cùng ACR registry**
- [ ] **Scheduled trigger** dùng **cron expression** để build theo lịch cố định
- [ ] **Multi-step task** định nghĩa bằng file **YAML**, cho phép build → test → push trong một pipeline
- [ ] `az acr run` để chạy container từ image có sẵn trong registry, dùng `/dev/null` khi không cần source file
- [ ] Luôn lưu PAT trong **Azure Key Vault**, không hardcode
- [ ] Dùng `.dockerignore` để giảm kích thước context upload
- [ ] Sắp xếp Dockerfile: instruction **ít thay đổi lên đầu** để tận dụng layer cache

---

[← Bài 1](./acr-m1-bai1-registries-repositories-artifacts.md) · [🏠 Mục lục](../README.md) · [Bài 3 →](./acr-m1-bai3-tag-and-version-images.md)
