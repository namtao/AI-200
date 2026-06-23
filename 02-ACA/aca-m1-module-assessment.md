# Module Assessment — Deploy Containers to Azure Container Apps

> Khoá: AI-200 · ACA — Deploy containers to Azure Container Apps

---

## Câu 1

**You're deploying an AI document-processing API that must access a private endpoint and share logs and networking settings across multiple container apps. Which Azure Container Apps resource provides that shared boundary?**

- A Container Apps environment ✅
- A Container Apps revision
- A replica

**Đáp án: A Container Apps environment**

Environment là **logical boundary** được chia sẻ bởi nhiều container app — chính xác là thứ bài toán đang cần. Apps trong cùng environment chia sẻ:
- **Virtual network** → cho phép giao tiếp nội bộ và truy cập private endpoint
- **Log integration** → tất cả app gửi log về cùng Log Analytics workspace
- **Governance** → policy áp dụng ở cấp environment

Tại sao các đáp án còn lại sai:
- **Revision** là versioned snapshot của config một app cụ thể — nó không shared giữa nhiều app và không quản lý networking hay logging.
- **Replica** là running instance của một revision — đây là đơn vị compute nhỏ nhất, không phải boundary cho shared config.

---

## Câu 2

**You want a consistent, source-controlled deployment for a container app so that configuration is reviewed like code. Which approach best supports that goal?**

- Use `az containerapp create --yaml` and `az containerapp update --yaml` with a YAML file stored in source control ✅
- Use `az containerapp update --set-env-vars` for all changes without maintaining a configuration file
- Rely on image tags and redeploy with `az containerapp update --image` to apply environment-specific configuration

**Đáp án: Use `--yaml` with a YAML file stored in source control**

YAML-based deployment là approach duy nhất trong danh sách đáp ứng cả hai yêu cầu: **consistent** (cùng file → cùng kết quả) và **source-controlled** (file trong Git → thay đổi config đi qua Pull Request như code).

```bash
# Deploy từ YAML — source of truth duy nhất
az containerapp create -n ai-api -g rg-aca-demo \
    --environment aca-env-demo \
    --yaml ./containerapp.yml

az containerapp update -n ai-api -g rg-aca-demo \
    --yaml ./containerapp.yml
```

Khi dùng `--yaml`, tất cả CLI flag khác bị bỏ qua — YAML là nguồn duy nhất.

Tại sao các đáp án còn lại sai:
- **`--set-env-vars` không có file** → config tồn tại trong state của Azure, không có artifact trong source control để review hay reproduce. Mỗi lần thay đổi là imperative command, không có history dễ đọc.
- **Image tag + `--image`** → chỉ kiểm soát version của image, không phải toàn bộ configuration (env vars, secrets, scaling, ingress...). Không đủ để là "configuration reviewed like code".

---

## Câu 3

**Your container app needs an API key for an embeddings provider. You don't want to store the value in a YAML file. Which pattern should you use?**

- Store the secret in Container Apps secrets and reference it from an environment variable ✅
- Add the API key as plain text in the YAML file under `env:`
- Bake the API key into the container image

**Đáp án: Store in Container Apps secrets và reference từ environment variable**

Đây chính xác là pattern `secretref:` được thiết kế cho tình huống này:

```bash
# Bước 1: Lưu API key vào secret store
az containerapp secret set -n ai-api -g rg-aca-demo \
    --secrets embeddings-api-key="sk-abc123..."

# Bước 2: Map secret vào env var bằng secretref
az containerapp update -n ai-api -g rg-aca-demo \
    --set-env-vars EMBEDDINGS_API_KEY=secretref:embeddings-api-key
```

YAML chỉ lưu **tên secret**, không lưu giá trị:
```yaml
- name: EMBEDDINGS_API_KEY
  secretRef: embeddings-api-key    # tên secret, không phải giá trị
```

App đọc `EMBEDDINGS_API_KEY` như env var bình thường — không biết giá trị đến từ secret store.

Tại sao các đáp án còn lại sai:
- **Plain text trong YAML** vi phạm yêu cầu của câu hỏi ("không muốn lưu trong YAML"). Ngoài ra, nếu YAML được commit lên Git, API key bị lộ trong source control history vĩnh viễn.
- **Bake vào container image** là cách tệ nhất: mọi người có quyền pull image đều thấy được key, và mỗi lần rotate key phải rebuild image và redeploy toàn bộ.

---

## Câu 4

**You updated a container app and want to confirm the new configuration is running in production. Which command helps you see the versioned change that Container Apps created?**

- Run `az containerapp revision list` ✅
- Run `az containerapp registry list`
- Run `az containerapp secret list`

**Đáp án: `az containerapp revision list`**

Mỗi khi update container app, ACA tạo một **revision mới** — đây là versioned snapshot của config tại thời điểm đó. Liệt kê revision cho thấy:
- Revision mới đã được tạo chưa
- `healthState` của revision (Healthy/Unhealthy)
- Revision nào đang active và nhận traffic (`trafficWeight`)

```bash
az containerapp revision list -n ai-api -g rg-aca-demo

# Bao gồm cả inactive revision
az containerapp revision list -n ai-api -g rg-aca-demo --all
```

Tại sao các đáp án còn lại sai:
- **`registry list`** liệt kê registry được cấu hình cho app — không liên quan đến version của deployment.
- **`secret list`** liệt kê tên secret (không phải giá trị) — cũng không liên quan đến việc confirm deployment version.

---

## Câu 5

**A container app fails to start after you update the image. You need fast feedback to diagnose the issue. Which command is the best first step?**

- Run `az containerapp logs show` ✅
- Run `az containerapp revision list`
- Run `az containerapp replica list`

**Đáp án: `az containerapp logs show`**

Log là **nguồn thông tin nhanh nhất** khi container fail to start. Startup log thường hiện rõ nguyên nhân:
- Missing environment variable
- Port binding error (app bind vào localhost thay vì 0.0.0.0)
- Connection failure đến dependency
- Image pull error (nếu là system log)
- Authentication failure khi gọi model provider

```bash
# Xem log ngay
az containerapp logs show -n ai-api -g rg-aca-demo

# Follow real-time nếu cần
az containerapp logs show -n ai-api -g rg-aca-demo --follow --tail 50

# System log nếu nghi ngờ platform issue
az containerapp logs show -n ai-api -g rg-aca-demo --type system
```

Tại sao các đáp án còn lại sai:
- **`revision list`** cho biết revision có tồn tại và healthy không — nhưng không cho biết **tại sao** fail. Biết revision Unhealthy chưa đủ để fix.
- **`replica list`** cho biết instance có đang chạy không — nhưng tương tự, không có error detail. Khi container fail to start, replica list chỉ cho thấy replica không running, không giải thích nguyên nhân.

Quy trình đúng: **log trước** → nếu không đủ thông tin → mới kiểm tra revision và replica.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Shared boundary cho networking và logging | Container Apps environment |
| 2 | Source-controlled, reviewable deployment | `--yaml` với file trong source control |
| 3 | Sensitive config không lưu trong YAML | Container Apps secrets + `secretref:` |
| 4 | Confirm versioned change sau khi update | `az containerapp revision list` |
| 5 | First step khi container fail to start | `az containerapp logs show` |

---

*Module hoàn thành. Learning Path ACA tiếp theo: Module 2*
