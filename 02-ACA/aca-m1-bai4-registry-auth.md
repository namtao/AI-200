# Bài 4 — Configure Image Pull Authentication for Private Registries

> Khoá: AI-200 · ACA — Deploy containers to Azure Container Apps

---

## Tại sao cần private registry?

Production AI service thường pull image từ **private registry** thay vì public Docker Hub, vì:

- **Supply chain control:** Chỉ dùng image đã được team approve, không pull image random từ internet
- **Vulnerability scanning:** Quét bảo mật image trước khi deploy
- **Access control:** Kiểm soát ai được push và pull image

Private registry yêu cầu xác thực — ACA cần credential hoặc identity để pull image. Bài này giải thích hai cách cấu hình.

> **Lưu ý debug quan trọng:** Registry configuration **tách biệt** với application configuration. Nếu app fail sau khi thay đổi registry config, kiểm tra container log — image pull failure thường trông giống authentication error hoặc permission error.

---

## Hai cách xác thực với private registry

| | Username & Password | Managed Identity |
|---|---|---|
| Hoạt động với | Mọi registry | Chỉ Azure Container Registry |
| Cần lưu credential | ✅ Phải lưu password | ❌ Không cần |
| Rotation overhead | ⚠️ Phải rotate thủ công | ❌ Azure tự quản lý |
| Setup complexity | Thấp | Cần role assignment trước |
| Phù hợp | Quick validation, non-ACR registry | Production với ACR |

---

## Cách 1: Username và Password

Cách đơn giản hơn — cung cấp server URL + username + password trực tiếp.

```bash
az containerapp registry set \
    -n ai-api \
    -g rg-aca-demo \
    --server myregistry.azurecr.io \
    --username MyRegistryUsername \
    --password MyRegistryPassword
```

Lệnh `registry set` gắn registry config vào container app — các lần pull image (khi tạo revision mới, scale out instance mới...) đều dùng credential này.

**Khi nào dùng username/password:**
- Registry không phải ACR (Docker Hub private, GitHub Container Registry, self-hosted)
- Quick validation trong development
- Testing registry config trước khi chuyển sang managed identity

**Nhược điểm trong production:**
- Password là long-lived credential — nếu lộ, phải rotate tay ở tất cả app đang dùng
- Password xuất hiện trong deployment script → dễ bị commit lên Git nhầm
- Khi nhiều AI service cùng dùng một registry, rotation credential trở thành gánh nặng vận hành lớn

> **Với ACR:** CLI đôi khi tự infer credential nếu bỏ `--username` và `--password`. Tuy nhiên trong production nên luôn explicit để behavior predictable — không phụ thuộc vào auto-detection.

---

## Cách 2: Managed Identity (Khuyến nghị cho ACR + production)

Managed identity cho phép ACA xác thực với ACR **mà không cần lưu credential nào**. Azure tự quản lý identity và token rotation.

```bash
az containerapp registry set \
    -n ai-api \
    -g rg-aca-demo \
    --server myregistry.azurecr.io \
    --identity system
```

`--identity system` = dùng system-assigned managed identity của container app.

### Yêu cầu trước khi dùng managed identity

Trước khi chạy lệnh trên, cần đảm bảo:

1. **Managed identity đã được enable** trên container app (system-assigned hoặc user-assigned)
2. **Identity được gán role `AcrPull`** trên ACR registry

Nếu chưa gán role, ACA pull image sẽ bị từ chối dù lệnh `registry set` thành công.

### System-assigned vs User-assigned identity

- `--identity system` → dùng system-assigned identity của app
- `--identity <resource-id>` → dùng user-assigned identity cụ thể (theo resource ID)

Xem lại bài acr-m2-bai1 để nhắc lại sự khác biệt: system-assigned gắn với vòng đời app, user-assigned tồn tại độc lập và có thể dùng chung cho nhiều app.

### Tại sao managed identity phù hợp cho production AI system?

AI backend thường có nhiều service cùng pull từ một registry. Với username/password, rotate credential đồng nghĩa với cập nhật config của từng service một. Với managed identity, tất cả service dùng identity của mình — không có credential nào để rotate.

---

## Verify registry config

Khi debug image pull failure, bước đầu tiên là xác nhận registry đã được cấu hình đúng.

### Liệt kê registry được cấu hình

```bash
az containerapp registry list \
    -n ai-api \
    -g rg-aca-demo
```

Dùng khi: muốn biết app đang dùng registry nào, phát hiện drift giữa các environment.

### Xem chi tiết một registry cụ thể

```bash
az containerapp registry show \
    -n ai-api \
    -g rg-aca-demo \
    --server myregistry.azurecr.io
```

Dùng khi: verify server URL, authentication method, và identity đang được dùng.

---

## Best Practices

### Dùng managed identity cho ACR

```bash
# Production — managed identity, không credential
az containerapp registry set -n ai-api -g rg-aca-demo \
    --server myregistry.azurecr.io \
    --identity system

# Development/non-ACR — username/password khi cần
az containerapp registry set -n ai-api -g rg-aca-demo \
    --server ghcr.io \
    --username myuser \
    --password $GITHUB_TOKEN
```

### Chỉ cấp quyền tối thiểu — AcrPull

Identity dùng để pull image chỉ cần role `AcrPull`, không cần `AcrPush` hay `Owner`. Principle of least privilege: nếu identity bị compromise, attacker chỉ pull được image, không push được image độc hại vào registry.

### Dùng fully qualified image name

```bash
# Tốt — rõ ràng, không ambiguous
--image myregistry.azurecr.io/ai-api:v1

# Tránh — Docker Hub được assume, dễ nhầm
--image ai-api:v1
```

Fully qualified name bao gồm registry hostname giúp tránh nhầm lẫn khi có nhiều registry.

### Validate authentication ngay khi thay đổi

Sau khi cập nhật registry config, deploy revision mới và verify revision đó start thành công trước khi kết luận config đúng. Image pull failure chỉ xảy ra khi ACA thực sự cố gắng pull — không phải khi set config.

---

## Bản chất bài này là gì?

**Một câu:** Bài này giống hệt ACR bài 1 M2 về concept — ACA cần credential để pull image từ private registry, và managed identity thay thế `docker login` trên cloud.

### So sánh

| | Docker local | App Service | ACA |
|---|---|---|---|
| Xác thực với registry | `docker login registry.io` | Managed identity hoặc admin creds | Managed identity hoặc username/password |
| Lưu credential | Trong `~/.docker/config.json` | App config (encrypted) | Trong container app config |
| Không cần credential | Không | ✅ managed identity với ACR | ✅ `--identity system` |
| Xác thực với non-ACR registry | `docker login` | username/password | `registry set --username --password` |

**Điểm giống App Service hoàn toàn:** Cùng khái niệm, cùng lệnh pattern, chỉ khác CLI prefix (`az webapp` vs `az containerapp`). Managed identity → `AcrPull` role. Không có credential nào cần lưu hay rotate.

---

## Checklist ghi nhớ cho AI-200

- [ ] Registry config **tách biệt** với app config — image pull failure trông như auth error trong log
- [ ] Hai cách xác thực: **username/password** (mọi registry) và **managed identity** (chỉ ACR)
- [ ] Cấu hình registry: `az containerapp registry set --server ... --username ... --password ...`
- [ ] Managed identity: `az containerapp registry set --server ... --identity system`
- [ ] Managed identity yêu cầu identity được gán role **`AcrPull`** trên ACR trước
- [ ] `--identity system` = system-assigned · `--identity <resource-id>` = user-assigned
- [ ] Managed identity phù hợp production vì không có long-lived credential cần rotate
- [ ] Verify registry: `az containerapp registry list` và `az containerapp registry show`
- [ ] Chỉ gán role **`AcrPull`** cho pull identity — không cần AcrPush
- [ ] Luôn dùng **fully qualified image name** bao gồm registry hostname

---

[← Bài 3](./aca-m1-bai3-runtime-settings.md) · [🏠 Mục lục](../README.md) · [Bài 5 →](./aca-m1-bai5-verify-deployments.md)
