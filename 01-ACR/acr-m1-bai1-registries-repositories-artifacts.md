# Bài 1 — Registries, Repositories, và Artifacts (Azure Container Registry)

> Khoá: AI-200 · Module: Implement container application hosting on Azure

---

## Vấn đề ACR giải quyết

Khi phát triển AI application với container, bạn cần một nơi để lưu trữ và phân phối container image. Có hai lựa chọn phổ biến:

- **Docker Hub** — registry công khai, mặc định ai cũng thấy được image của bạn
- **Azure Container Registry (ACR)** — registry riêng tư trên Azure, chỉ người được cấp quyền mới truy cập được

Với AI application, việc giữ image riêng tư là rất quan trọng vì image thường chứa code xử lý model, cấu hình hệ thống, và các thông tin nhạy cảm khác.

---

## Azure Container Registry (ACR) là gì?

ACR là một **managed private Docker registry** do Azure cung cấp — "managed" nghĩa là Azure lo phần vận hành hạ tầng, bạn chỉ cần dùng.

Nói đơn giản: ACR là "kho lưu trữ" container image của riêng bạn trên Azure, tương tự Docker Hub nhưng private và tích hợp chặt với hệ sinh thái Azure.

### Tại sao dùng ACR thay vì Docker Hub cho AI workloads?

**Private storage — lưu trữ riêng tư**

Toàn bộ image của bạn — model-serving container, preprocessing container, inference API — đều được giữ trong môi trường Azure riêng tư. Không ai bên ngoài có thể pull về trừ khi bạn cấp quyền.

**Geo-replication — nhân bản theo vùng địa lý**

Nếu bạn deploy AI application ở nhiều region (ví dụ: Southeast Asia và West Europe), ACR có thể tự động nhân bản image đến cả hai region. Khi Kubernetes cluster ở Europe cần pull image, nó lấy từ bản gần nhất thay vì phải kéo từ tận region khác — giảm đáng kể thời gian deploy.

**Tích hợp trực tiếp với Azure services**

ACR kết nối sẵn với Azure Kubernetes Service (AKS), Azure Container Apps, và Azure App Service. Bạn không cần cấu hình authentication thêm — các service này tự biết cách pull image từ ACR trong cùng subscription.

**Đa định dạng nội dung**

ACR không chỉ lưu Docker image. Trong cùng một registry, bạn có thể lưu:
- Docker image (container image thông thường)
- Helm chart (package định nghĩa Kubernetes deployment)
- OCI artifact (bất kỳ nội dung tuân theo chuẩn OCI — Open Container Initiative)

### Các tier dịch vụ

ACR có 3 tier với tính năng và chi phí khác nhau:

| Tier | Phù hợp với | Điểm khác biệt |
|---|---|---|
| Basic | Học tập, phát triển cá nhân | Storage và throughput giới hạn, đủ để thực hành |
| Standard | Production thông thường | Thêm storage và throughput so với Basic |
| Premium | Enterprise, triển khai lớn | Geo-replication, private endpoint, content trust |

> **Quan trọng cho thi AI-200:** Geo-replication và content trust **chỉ có ở Premium tier**. Đây là điểm phân biệt thường ra trong đề thi.

**Giải thích các tính năng Premium:**
- **Geo-replication:** Tự động nhân bản image đến nhiều Azure region
- **Private endpoint:** Kết nối ACR qua Azure Private Link, không qua internet công cộng — tăng bảo mật
- **Content trust:** Ký số (digital signing) cho image, đảm bảo image không bị giả mạo sau khi push

Hầu hết workflow developer hoạt động giống nhau trên cả 3 tier — sự khác biệt chủ yếu ở tính năng enterprise và giới hạn tài nguyên.

---

## Cấu trúc phân cấp 3 tầng

ACR tổ chức nội dung theo 3 tầng lồng nhau. Hiểu rõ cấu trúc này giúp bạn thiết kế registry hợp lý cho team.

```
Registry  (myregistry.azurecr.io)
├── Repository  (inference-api)
│   ├── Artifact  :v1.0.0
│   ├── Artifact  :v1.2.0
│   └── Artifact  :latest
└── Repository  (model-server)
    └── Artifact  :v2.0.0
```

### Tầng 1: Registry

Registry là tài nguyên cấp cao nhất, chứa toàn bộ nội dung container. Mỗi registry có một **login server URL duy nhất** theo định dạng:

```
<tên-registry>.azurecr.io
```

Ví dụ: nếu bạn đặt tên registry là `contosoinference` thì login server sẽ là `contosoinference.azurecr.io`.

Registry chịu trách nhiệm:
- **Authentication** — xác thực ai được phép truy cập
- **Access control** — ai được push, ai chỉ được pull
- **Retention policy** — tự động xoá image cũ theo quy tắc
- **Geo-replication** — nhân bản đến nhiều region (Premium)

Trong một tổ chức, thường có một registry chung cho toàn bộ team, hoặc mỗi nhóm/dự án có registry riêng tùy quy mô.

### Tầng 2: Repository

Repository là **tập hợp các image có cùng tên nhưng khác tag**. Ví dụ, repository `inference-api` chứa tất cả các phiên bản của image đó: `v1.0.0`, `v1.2.0`, `latest`,...

Mỗi lần bạn push một image mới với tag khác, nó xuất hiện trong cùng repository đó.

#### Namespace — tổ chức repository theo nhóm

Repository hỗ trợ namespace dùng dấu `/` để tạo cấu trúc phân cấp logic:

```
production/inference-api    → image đã qua kiểm duyệt, sẵn sàng production
staging/inference-api       → image đang được kiểm tra
ml-team/model-server        → image của team ML
devops/base-python          → base image do team DevOps quản lý
```

Namespace không chỉ giúp tổ chức gọn gàng mà còn phục vụ **phân quyền**. Bạn có thể cấp cho team ML quyền push vào `ml-team/` nhưng không cho phép họ push vào `production/`. Điều này ngăn việc vô tình ghi đè image production.

### Tầng 3: Artifact

Artifact là **nội dung thực tế** được lưu trong repository — thường là Docker image, nhưng cũng có thể là Helm chart hay OCI artifact khác.

Mỗi artifact gồm 3 thành phần: **tag**, **layer**, và **manifest** — sẽ được giải thích chi tiết ở phần tiếp theo.

---

## Tags, Layers, và Manifests — 3 thành phần của một Artifact

### Tag — nhãn định danh, có thể thay đổi

Tag là nhãn người đọc được dùng để xác định một phiên bản cụ thể của artifact. Định dạng đầy đủ là:

```
registry/repository:tag
```

Ví dụ: `myregistry.azurecr.io/inference-api:v1.2.0`

Một số điều quan trọng về tag:

**Một image có thể có nhiều tag**

Bạn có thể gán nhiều tag cho cùng một image. Ví dụ, sau khi kiểm tra xong `v1.2.0` và quyết định đây là bản stable:

```bash
# Cả hai tag này trỏ đến cùng một image
myregistry.azurecr.io/inference-api:v1.2.0
myregistry.azurecr.io/inference-api:stable
```

**Tag là mutable — có thể bị thay đổi**

Đây là điểm cực kỳ quan trọng cần nhớ. Khi bạn push một image mới với tag đã tồn tại, tag đó sẽ **chuyển sang trỏ vào image mới**. Image cũ vẫn còn trong registry nhưng không có tag nào trỏ vào nữa (trở thành untagged manifest).

Điều này có nghĩa là: nếu bạn deploy với tag `latest` hôm nay, ngày mai ai đó push image mới cũng với tag `latest`, thì khi bạn scale up thêm node mới — node mới sẽ pull phiên bản khác với node cũ. Đây là nguồn gốc của nhiều lỗi khó debug trong production.

**Tag mặc định là `latest`**

Nếu bạn `docker push` hoặc `docker pull` mà không chỉ định tag, Docker tự động dùng tag `latest`. Đây là convention, không phải tính năng đặc biệt — `latest` không nhất thiết là phiên bản mới nhất nếu bạn không push đúng cách.

---

### Layer — các lớp cấu thành image

Container image không phải một file đơn lẻ — nó được tạo thành từ nhiều **layer** xếp chồng lên nhau. Mỗi layer tương ứng với một lệnh trong Dockerfile làm thay đổi filesystem.

Ví dụ Dockerfile đơn giản:

```dockerfile
FROM python:3.11          # Layer 1: lấy base image python
RUN pip install torch     # Layer 2: cài thư viện torch
COPY . /app               # Layer 3: copy source code vào
CMD ["python", "app.py"]  # Không tạo layer mới (chỉ là metadata)
```

Image cuối cùng có 3 layer xếp chồng lên nhau.

**Layer deduplication — chia sẻ layer chung**

Đây là cơ chế quan trọng giúp tiết kiệm storage. Mỗi layer được định danh bằng hash của nội dung. Nếu hai image dùng cùng một layer (ví dụ cùng `FROM python:3.11`), ACR chỉ lưu một bản duy nhất và cả hai image cùng tham chiếu đến bản đó.

Với AI application, điều này đặc biệt có giá trị: bạn có thể có 10 image khác nhau (inference API, preprocessing, training job...) nhưng tất cả đều dùng chung `python:3.11` và `torch` — ACR chỉ lưu các layer đó một lần.

Layer cũng được cache khi build — nếu một layer không thay đổi, Docker/ACR dùng lại bản cache thay vì build lại từ đầu, giúp build nhanh hơn đáng kể.

---

### Manifest — bản mô tả artifact, không thay đổi

Mỗi artifact có một **manifest** là file JSON mô tả toàn bộ cấu trúc của artifact đó: danh sách các layer theo thứ tự, cấu hình image (biến môi trường, entrypoint, v.v.), và metadata khác.

Manifest được định danh bằng **SHA-256 digest** — một chuỗi hash 64 ký tự:

```
sha256:0a2e01852872580b2c2fea9380ff8d7b637d3928783c55beb3f21a6e58d5d108
```

**Digest là immutable — không bao giờ thay đổi**

Đây là điểm khác biệt cốt lõi so với tag. Một khi image đã được push và có digest, digest đó vĩnh viễn gắn với image đó. Không có cách nào thay đổi digest của một image đã push — muốn thay đổi thì phải push image mới và sẽ có digest mới.

Điều này tạo ra **tính bất biến** (immutability) — nếu bạn biết digest, bạn có thể chắc chắn 100% rằng mình sẽ lấy đúng image đó, dù bao nhiêu thời gian trôi qua hay tag có thay đổi.

---

## Hai cách địa chỉ hóa image khi pull/push

### Cách 1: Theo tag — dễ đọc, dùng trong development

```bash
# Pull image theo tag
docker pull myregistry.azurecr.io/inference-api:v1.2.0

# Push image với tag
docker push myregistry.azurecr.io/inference-api:v1.2.0
```

Tag-based addressing dễ nhớ và dễ đọc. Phù hợp khi:
- Phát triển local và muốn thử phiên bản mới nhất
- Dùng tag như `nightly` hoặc `dev` để nhận cập nhật tự động
- Môi trường development và staging nơi tính nhất quán tuyệt đối không cần thiết

Nhược điểm: vì tag có thể thay đổi, deployment dùng tag không đảm bảo reproducibility.

### Cách 2: Theo digest — bất biến, dùng trong production

```bash
# Pull image theo digest
docker pull myregistry.azurecr.io/inference-api@sha256:0a2e01852872580b2c2fea9380ff8d7b637d3928783c55beb3f21a6e58d5d108
```

Digest-based addressing đảm bảo bạn **luôn lấy đúng image đó**, bất kể tag có bị thay đổi hay không.

Bạn có thể tìm digest của một image tại:
- Azure Portal → registry → repository → chọn image → xem digest
- Azure CLI: `az acr repository show-manifests --name myregistry --repository inference-api`
- Output khi push image: Docker in ra digest ngay sau khi push thành công

> **Nguyên tắc quan trọng cho production AI:** Khi deploy AI model lên Kubernetes cluster với nhiều node, tất cả node cần chạy đúng cùng một image. Nếu dùng tag và tag bị thay đổi giữa chừng, node mới scale lên sẽ pull phiên bản khác với node cũ — gây ra hành vi không nhất quán rất khó debug. Luôn dùng **digest** trong production deployment manifest.

---

## Best Practices tổ chức Registry

### Lên kế hoạch cấu trúc namespace từ đầu

Trước khi bắt đầu push image, hãy quyết định cấu trúc namespace. Thay đổi sau khi đã có nhiều image rất tốn công. Một cấu trúc phổ biến:

```
<environment>/<service-name>
production/inference-api
staging/inference-api
dev/inference-api

<team>/<service-name>
ml-team/feature-extractor
data-team/preprocessing
```

### Dùng namespace để phân quyền

Gán quyền theo namespace thay vì theo registry để giảm thiểu rủi ro:
- Team dev: push/pull `dev/*` và `staging/*`, chỉ pull `production/*`
- CI/CD pipeline: push `production/*` sau khi pass đủ test
- Read-only service account: chỉ pull image, không push được

### Bật geo-replication cho global AI deployment

Nếu AI service của bạn chạy ở nhiều region, bật geo-replication (Premium) để giảm thời gian pull. Kéo image từ region gần nhất thay vì từ region khác có thể giảm deploy time từ vài phút xuống vài giây.

### Đặt retention policy và dọn dẹp định kỳ

Image tích lũy theo thời gian. Mỗi lần CI/CD chạy tạo ra một image mới. Sau vài tháng, bạn có thể có hàng trăm image không dùng đến.

Untagged manifest (image không có tag nào trỏ vào) là loại chiếm storage nhưng không có giá trị sử dụng. Đặt retention policy để tự động xoá chúng sau N ngày, hoặc chạy định kỳ lệnh dọn dẹp.

---

## So sánh ACR với Docker Hub

Vì nhiều khái niệm của ACR giống Docker (registry, repository, tag, layer, manifest), bảng dưới đây tập trung vào điểm khác biệt:

| Tính năng | Docker Hub | ACR |
|---|---|---|
| Registry / Repo / Tag | ✅ | ✅ Giống hệt |
| Layer & deduplication | ✅ | ✅ Giống hệt |
| Manifest + digest | ✅ | ✅ Giống hệt |
| Lệnh `docker pull/push` | ✅ | ✅ Chỉ đổi URL |
| Private by default | ❌ Mặc định public | ✅ Private |
| Geo-replication | ❌ | ✅ Premium tier |
| Private endpoint (không qua internet) | ❌ | ✅ Premium tier |
| Content trust (ký số image) | ✅ có nhưng riêng biệt | ✅ Premium tier |
| Tích hợp Azure IAM | ❌ | ✅ |
| Kết nối trực tiếp AKS / Container Apps | ❌ cần cấu hình thêm | ✅ tích hợp sẵn |
| Retention policy tích hợp | ❌ | ✅ |

Kết luận: Nếu bạn đã quen Docker Hub, phần lớn kiến thức chuyển thẳng sang ACR. Chỉ cần tập trung học thêm phần Azure-specific: tier và tính năng tương ứng, cách tích hợp với AKS/Container Apps, và private endpoint.

---

## Checklist ghi nhớ cho AI-200

- [ ] ACR là **private** Docker registry — khác Docker Hub ở chỗ mặc định không public
- [ ] **Tag = mutable** (có thể thay đổi), **Digest = immutable** (không bao giờ thay đổi)
- [ ] Production AI deployment nên dùng **digest** (`@sha256:...`) thay vì tag để đảm bảo nhất quán
- [ ] Cấu trúc 3 tầng: **Registry → Repository → Artifact**
- [ ] URL registry có dạng `<tên>.azurecr.io`
- [ ] Namespace dùng `/` phục vụ cả **tổ chức** lẫn **phân quyền**
- [ ] Layer deduplication: nhiều image dùng chung layer → ACR chỉ lưu một bản → tiết kiệm storage
- [ ] **Geo-replication và content trust chỉ có ở Premium tier**
- [ ] Private endpoint (kết nối không qua internet) cũng chỉ có ở **Premium**
- [ ] ACR tích hợp sẵn với AKS, Container Apps, App Service — không cần cấu hình thêm
- [ ] Untagged manifest tích lũy theo thời gian → cần retention policy để kiểm soát chi phí storage
- [ ] ACR hỗ trợ Docker image, Helm chart, và OCI artifact trong cùng một registry

---

[← Mục lục](../README.md) · [🏠 Mục lục](../README.md) · [Bài 2 →](./acr-m1-bai2-acr-tasks.md)
