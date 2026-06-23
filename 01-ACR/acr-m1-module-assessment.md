# Module Assessment — Store and Manage Containers in Azure Container Registry

> Khoá: AI-200 · Module: Implement container application hosting on Azure

---

## Câu 1

**Your team builds container images on developer workstations, leading to inconsistent results. You need to ensure all images are built in a controlled environment. Which Azure Container Registry feature addresses this requirement?**

- ACR Tasks quick build ✅
- Geo-replication
- Repository namespaces

**Đáp án: ACR Tasks quick build**

Vấn đề đặt ra là build trên máy developer cho kết quả không nhất quán ("works on my machine"). ACR Tasks giải quyết đúng vấn đề này bằng cách chuyển toàn bộ quá trình build lên Azure cloud — mọi build đều dùng cùng infrastructure, không phụ thuộc vào máy tính của developer nào.

Tại sao các đáp án còn lại sai:
- **Geo-replication** liên quan đến phân phối image đến nhiều Azure region để giảm độ trễ pull, không liên quan gì đến quá trình build.
- **Repository namespaces** chỉ là cách tổ chức tên repository (ví dụ `production/inference-api`), không ảnh hưởng đến môi trường build.

---

## Câu 2

**You need to deploy a container image to production and ensure every node in your Kubernetes cluster runs the exact same image version, even if someone pushes a new image with the same tag. How should you reference the image?**

- By manifest digest ✅
- By the latest tag
- By semantic version tag

**Đáp án: By manifest digest**

Yêu cầu là đảm bảo mọi node chạy đúng cùng một image, **kể cả khi ai đó push image mới đè lên cùng tag đó**. Chỉ có digest mới đảm bảo được điều này vì digest là **immutable** — một khi image đã được push và có digest `sha256:abc123...`, digest đó vĩnh viễn gắn với image đó, không có cách nào thay đổi.

```yaml
# Kubernetes manifest dùng digest — đảm bảo tuyệt đối
image: myregistry.azurecr.io/inference-api@sha256:0a2e01852872...
```

Tại sao các đáp án còn lại sai:
- **latest tag** là stable tag thay đổi mỗi khi có build mới — node khác nhau pull vào thời điểm khác nhau có thể nhận image khác nhau.
- **Semantic version tag** (ví dụ `v1.2.0`) là mutable — về mặt kỹ thuật, vẫn có thể push image khác đè lên tag đó. Tag không đảm bảo immutability, chỉ digest mới đảm bảo.

---

## Câu 3

**Your AI application depends on a base image containing PyTorch. When the PyTorch team releases security patches to the base image, you want your application image to rebuild automatically. Which ACR Tasks trigger type provides this capability?**

- Base image update trigger ✅
- Source code commit trigger
- Scheduled trigger

**Đáp án: Base image update trigger**

Base image update trigger được thiết kế chính xác cho tình huống này: ACR theo dõi câu lệnh `FROM` trong Dockerfile, và khi base image (PyTorch trong trường hợp này) được cập nhật, ACR tự động trigger rebuild application image — không cần can thiệp thủ công.

```dockerfile
FROM pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime  # ACR theo dõi image này
# Khi image trên được cập nhật → ACR tự rebuild image của bạn
```

Lưu ý: Base image trigger hoạt động tốt nhất khi cả base image và application image cùng trong một ACR registry. Với Docker Hub vẫn hoạt động nhưng có thể chậm hơn.

Tại sao các đáp án còn lại sai:
- **Source code commit trigger** chỉ phản ứng khi có commit/PR mới trong Git repo của bạn — không theo dõi base image bên ngoài.
- **Scheduled trigger** chạy theo lịch cố định (ví dụ mỗi đêm) — không phản ứng ngay khi base image có patch mới, mà phải chờ đến lịch tiếp theo.

---

## Câu 4

**You're implementing a tagging strategy for production deployments. Your requirements include traceability to the source code commit and the ability to roll back to any previous version. Which tagging pattern best meets these requirements?**

- Unique tags with Git commit SHA ✅
- Stable tags like v1 and v2
- Using only the latest tag

**Đáp án: Unique tags with Git commit SHA**

Yêu cầu có hai phần, và cần phân tích từng phần:

**Traceability đến source code commit** → Git commit SHA đáp ứng trực tiếp. Từ tag `inference-api:abc123f`, bất kỳ ai có quyền truy cập repo đều có thể chạy `git show abc123f` để xem chính xác code tạo ra image đó.

**Khả năng rollback về bất kỳ version nào** → Unique tag đáp ứng điều này. Vì tag không bao giờ bị ghi đè, mọi image lịch sử đều còn trong registry. Rollback chỉ cần đổi tag trong Kubernetes manifest về tag cũ.

```
inference-api:v1.2.0-abc123f   # Production hiện tại
inference-api:v1.1.0-def456g   # Rollback về đây nếu cần
inference-api:v1.0.0-ghi789h   # Vẫn còn, rollback về đây cũng được
```

Tại sao các đáp án còn lại sai:
- **Stable tags như v1, v2** bị ghi đè mỗi lần push — image cũ mất tag, không thể rollback về chính xác version đó.
- **latest tag** không có traceability (không biết đang chạy build nào) và không rollback được vì chỉ có một tag duy nhất luôn trỏ vào build mới nhất.

---

## Câu 5

**You deployed a critical AI inference API to production and need to prevent the container image from being accidentally deleted. Which ACR feature should you use?**

- Image locking with write-enabled false ✅
- Repository namespaces
- Geo-replication

**Đáp án: Image locking with write-enabled false**

```bash
az acr repository update \
  --name myregistry \
  --image inference-api:v1.2.0 \
  --write-enabled false
```

Sau khi lock, image có các đặc tính:
- **Không thể xoá** — kể cả admin cũng không xoá được trực tiếp
- **Không thể ghi đè** — push image mới với cùng tag sẽ bị từ chối
- **Không bị retention policy xoá** — script cleanup tự động bỏ qua image đã locked
- **Deployment ổn định** — production workload chắc chắn image luôn available

Khi muốn retire version đó, unlock trước rồi mới xoá:

```bash
az acr repository update \
  --name myregistry \
  --image inference-api:v1.2.0 \
  --write-enabled true
```

Tại sao các đáp án còn lại sai:
- **Repository namespaces** chỉ là cách tổ chức và phân quyền theo nhóm — không có cơ chế bảo vệ image khỏi bị xoá.
- **Geo-replication** nhân bản image đến nhiều region để giảm độ trễ — không liên quan đến việc ngăn xoá image.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | ACR Tasks — consistent build environment | ACR Tasks quick build |
| 2 | Immutable image reference cho production | Manifest digest |
| 3 | Tự động rebuild khi base image update | Base image update trigger |
| 4 | Tagging strategy — traceability + rollback | Unique tags with Git commit SHA |
| 5 | Bảo vệ image production khỏi bị xoá | Image locking (`--write-enabled false`) |

---

*Module hoàn thành. Bài tiếp theo: Exercise — Build and manage a container image with ACR Tasks*
