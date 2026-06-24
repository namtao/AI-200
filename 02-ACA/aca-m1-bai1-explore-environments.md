# Bài 1 — Explore Container Apps Environments

> Khoá: AI-200 · ACA — Deploy containers to Azure Container Apps

---

## ACA khác gì so với App Service?

Ở Learning Path trước, bạn deploy container lên **App Service** — phù hợp cho một web app hoặc API đơn giản. Azure Container Apps (ACA) hướng đến một bài toán khác: chạy **nhiều containerized service** cùng nhau, scale nhanh theo traffic, và tích hợp với nhau qua internal network — mà không cần quản lý Kubernetes.

Nói cách khác:
- **App Service** → một container, một web app, đơn giản
- **ACA** → nhiều container service, giao tiếp nội bộ, scale độc lập, không cần Kubernetes

Ví dụ scenario trong module: AI document-processing backend gồm nhiều service:
- API nhận upload từ client
- Service chạy text extraction
- Service gọi embeddings provider để tạo vector
- Background processor xử lý queue

Các service này cần giao tiếp với nhau, scale độc lập, và deploy an toàn từng bước — đây là lý do chọn ACA.

> **Lưu ý cài extension:** Các lệnh ACA cần Azure CLI extension. Cài hoặc upgrade bằng:
> ```bash
> az extension add --name containerapp --upgrade
> ```
> Nếu cần preview features, thêm `--allow-preview true`.

---

## Container Apps Environment là gì?

**Environment** là khái niệm trung tâm của ACA — quan trọng hơn ở App Service nhiều. Mọi container app đều phải thuộc về một environment.

Environment là **logical boundary** (ranh giới logic) nơi một hoặc nhiều container app chạy cùng nhau. Nó không chỉ là "folder" để nhóm resource — nó ảnh hưởng trực tiếp đến:

### 1. Networking — Mạng nội bộ

Các app trong cùng environment **chia sẻ một virtual network**. Điều này có nghĩa:
- App A có thể gọi App B bằng internal DNS name (không cần đi ra internet)
- Traffic nội bộ không tính phí egress
- Có thể giới hạn service chỉ nhận traffic từ trong environment (internal ingress)

App ở **khác environment** không thể giao tiếp nội bộ với nhau — phải đi qua public endpoint.

### 2. Observability — Theo dõi và log

Environment thường được tích hợp với **Azure Monitor** để tập trung log. Tất cả app trong environment gửi log về cùng một nơi — dễ query, dễ tạo alert, dễ trace request xuyên suốt nhiều service.

Với AI application, việc có log tập trung rất quan trọng vì nhiều lỗi là **data-dependent** — chỉ xảy ra với một số loại document hoặc input cụ thể, cần trace qua nhiều service để tìm nguyên nhân.

### 3. Governance — Quản trị đồng nhất

Các policy, access control, và cấu hình được áp dụng ở cấp environment — thay vì phải cấu hình từng app một.

---

## Tạo và kiểm tra Environment

### Tạo environment

Bạn có thể tạo environment trước (recommended khi nhiều app dùng chung), hoặc để `az containerapp up` tự tạo.

Tạo trước cho phép kiểm soát tên và lifecycle — quan trọng khi nhiều team dùng chung environment.

```bash
# Bước 1: Tạo resource group
az group create \
    --name rg-aca-demo \
    --location centralus

# Bước 2: Tạo environment
az containerapp env create \
    --name aca-env-demo \
    --resource-group rg-aca-demo \
    --location centralus
```

### Kiểm tra environment

Sau khi tạo, verify thông tin:

```bash
az containerapp env show \
    --name aca-env-demo \
    --resource-group rg-aca-demo
```

Dùng khi cần:
- Xác nhận app đang dùng environment nào
- Troubleshoot vấn đề region hoặc configuration
- Lấy thông tin như default domain, provisioning state

---

## Networking và Ingress — Ai có thể gọi được service của bạn?

Đây là quyết định quan trọng khi thiết kế AI solution vì nó ảnh hưởng trực tiếp đến bảo mật.

### External ingress — Công khai

Service nhận traffic từ internet. Dùng cho:
- Public-facing API mà web client gọi trực tiếp
- Webhook endpoint nhận event từ bên ngoài

```bash
az containerapp create \
    --name my-api \
    --ingress external \
    ...
```

### Internal ingress — Nội bộ

Service chỉ nhận traffic từ **các app trong cùng environment**. Không thể gọi từ internet. Dùng cho:
- Background processor chỉ được gọi bởi API service
- Embedding service chỉ phục vụ internal request
- Database proxy hoặc cache layer

```bash
az containerapp create \
    --name my-processor \
    --ingress internal \
    ...
```

**Tại sao internal ingress quan trọng với AI:**

AI backend thường có nhiều service không nên public. Ví dụ: embedding service xử lý dữ liệu nhạy cảm — chỉ nên gọi được từ API service của bạn, không nên expose ra internet. Internal ingress enforce điều này ở network level, không cần thêm authentication logic.

---

## Observability — Thu thập log ngay từ đầu

> "The first time you investigate a failed deployment is often when you realize you need consistent logging across apps."

Đây là lời khuyên thực tế từ tài liệu — đừng đợi đến khi có incident mới nghĩ đến logging.

ACA environment thường được tích hợp với **Azure Monitor Log Analytics**. Khi đã cấu hình, tất cả app trong environment tự động gửi log về — không cần setup riêng cho từng app.

Với AI application, log quan trọng vì:
- Lỗi thường là data-dependent — không reproduce được dễ dàng, phải trace qua log
- Request đi qua nhiều service — cần correlate log để hiểu toàn bộ flow
- Startup failure của container AI thường có nhiều chi tiết trong log (model loading, dependency check...)

---

## Best Practices cho Environment Design

### Tách environment theo lifecycle

```
aca-env-dev       → development
aca-env-test      → testing / staging
aca-env-prod      → production
```

Mỗi environment là một **blast radius boundary** — lỗi trong dev không ảnh hưởng production. Khi deploy revision mới, test trong dev/test environment trước.

### Gom service theo networking boundary

Các service **cần giao tiếp nội bộ** với nhau → đặt trong cùng environment.
Các service thuộc **team khác nhau, không liên quan** → đặt trong environment riêng.

```
aca-env-ai-backend:
    ├── document-api         (external ingress — nhận upload từ client)
    ├── text-extractor       (internal ingress — chỉ api gọi)
    └── embedding-service    (internal ingress — chỉ api gọi)

aca-env-monitoring:
    └── metrics-collector    (tách riêng — khác team quản lý)
```

### Naming convention nhất quán

Naming convention tốt giúp tooling tự động discover resource và giảm nhầm lẫn:

```
rg-<project>-<env>          resource group:    rg-docprocessor-prod
aca-env-<project>-<env>     environment:       aca-env-docprocessor-prod
<service>-<env>             app:               document-api-prod
```

### Lên kế hoạch observability trước khi deploy production

Quyết định trước:
- Log chảy về Log Analytics workspace nào?
- Alert được tạo cho loại lỗi nào?
- Dashboard nào cần build cho on-call team?

---

## So sánh ACA Environment với App Service Plan

| | App Service Plan | ACA Environment |
|---|---|---|
| Là gì | Compute tier, xác định CPU/RAM/pricing | Networking + observability boundary |
| Nhiều app trong một | ✅ nhưng dùng chung compute | ✅ mỗi app scale độc lập |
| Internal networking | ❌ không có | ✅ apps giao tiếp qua internal DNS |
| Log tập trung | ❌ cần cấu hình riêng | ✅ tích hợp sẵn với Azure Monitor |
| Kubernetes underneath | ❌ | ✅ (managed, bạn không thấy) |

---

## Bản chất bài này là gì?

**Một câu:** Environment là "Kubernetes namespace được quản lý" — ranh giới network, observability, và governance cho một nhóm microservice cùng chạy với nhau.

App Service Plan chỉ là **compute tier** (CPU/RAM). ACA Environment là thứ khác hẳn: nó quyết định **ai nói chuyện được với ai** và **log của ai đổ về đâu**.

### So sánh với các giải pháp khác

| Tính năng | App Service Plan | Docker Compose | Kubernetes | ACA Environment |
|---|---|---|---|---|
| Internal DNS giữa services | ❌ | ✅ service name | ✅ cluster DNS | ✅ tự động |
| Scale mỗi service độc lập | ❌ shared plan | ❌ manual | ✅ | ✅ |
| Log tập trung | ❌ cần cấu hình riêng | ❌ | ❌ cần setup | ✅ tích hợp sẵn |
| Quản lý infrastructure | Bạn chọn tier | Bạn lo host | Bạn lo k8s cluster | Azure lo |

**Mental model:** Một Environment ≈ một `docker-compose` network scope — services trong cùng network gọi nhau bằng tên, services ngoài network không nhìn thấy nhau. Nhưng trên cloud và có auto-scaling.

**Điểm quan trọng nhất:** Apps ở **khác environment** = không giao tiếp nội bộ được, dù cùng subscription.

---

## Checklist ghi nhớ cho AI-200

- [ ] Mọi container app đều phải thuộc về một **environment**
- [ ] Environment ảnh hưởng đến: **networking** (internal traffic), **observability** (log tập trung), **governance** (policy chung)
- [ ] Cài ACA CLI extension: `az extension add --name containerapp --upgrade`
- [ ] Tạo environment: `az containerapp env create --name ... --resource-group ... --location ...`
- [ ] Apps trong cùng environment **chia sẻ virtual network** → có thể giao tiếp nội bộ
- [ ] Apps ở **khác environment** không thể giao tiếp nội bộ
- [ ] **External ingress** → public, nhận traffic từ internet
- [ ] **Internal ingress** → chỉ nhận traffic từ app trong cùng environment
- [ ] Tách environment theo lifecycle: dev / test / prod
- [ ] Lên kế hoạch observability (Log Analytics) **trước** khi deploy production traffic
- [ ] Naming convention nhất quán giúp tooling tự động discover resource

---

*Bài tiếp theo: Bài 2 — Deploy a container app*
