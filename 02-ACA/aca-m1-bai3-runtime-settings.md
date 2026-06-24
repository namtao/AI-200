# Bài 3 — Configure Runtime Settings with Environment Variables and Secrets

> Khoá: AI-200 · ACA — Deploy containers to Azure Container Apps

---

## Tổng quan

AI service thường phụ thuộc vào nhiều external system và cần nhiều runtime setting: endpoint URL, API key, feature flag, model selection, timeout, batch size... Bài này giải thích cách quản lý hai loại setting:

```
Runtime settings
├── Environment variables  → non-sensitive config (log level, URL, feature flag...)
└── Secrets               → sensitive config (API key, password, signing key...)
    └── Secret references → map secret vào environment variable (secretref:)
```

---

## 1. Environment Variables — Non-sensitive config

Environment variable phù hợp cho các setting **không nhạy cảm** và thường thay đổi giữa các environment:
- Log level (`LOG_LEVEL=info` ở production, `LOG_LEVEL=debug` khi debug)
- Feature flags (`FEATURE_EMBEDDINGS=true/false`)
- Dependency URLs (`STORAGE_ENDPOINT=https://...`)
- Tuning parameters (`REQUEST_TIMEOUT=30`, `BATCH_SIZE=100`)

Cách tiếp cận này giữ container image **portable** — cùng một image chạy ở dev, staging, production với config khác nhau, không cần rebuild.

### Set env vars khi tạo app

```bash
az containerapp create \
    -n ai-api \
    -g rg-aca-demo \
    --environment aca-env-demo \
    --image myregistry.azurecr.io/ai-api:v1 \
    --ingress external \
    --target-port 8000 \
    --env-vars LOG_LEVEL=info FEATURE_EMBEDDINGS=true
```

`--env-vars` nhận danh sách cặp `KEY=VALUE` cách nhau bằng dấu cách. Có thể set nhiều biến cùng lúc.

### Thêm hoặc cập nhật env vars sau khi tạo

```bash
az containerapp update \
    -n ai-api \
    -g rg-aca-demo \
    --set-env-vars LOG_LEVEL=debug
```

`--set-env-vars` (khác với `--env-vars`) chỉ **thêm hoặc cập nhật** biến được chỉ định — các biến hiện có khác không bị xoá. Đây là cách an toàn hơn khi chỉ muốn thay đổi một vài biến mà không ảnh hưởng phần còn lại.

> **Phân biệt `--env-vars` vs `--set-env-vars`:**
> - `--env-vars` (dùng với `create`): set env vars khi tạo app
> - `--set-env-vars` (dùng với `update`): thêm/cập nhật env vars, giữ nguyên các biến khác

---

## 2. Secrets — Sensitive config

Secret là cách lưu trữ **giá trị nhạy cảm** tách biệt khỏi container image và source control. Với AI service, đây bao gồm:
- Provider API key (OpenAI, Azure AI...)
- Database credentials
- Signing key, encryption key
- Webhook secret

### Tại sao cần tách secret ra riêng?

**Rotation không cần redeploy:** Key bị lộ hoặc hết hạn → chỉ update secret, không cần rebuild image hay redeploy toàn bộ app.

**Không lưu trong source control:** Secret value không xuất hiện trong Dockerfile, Git history, hay CI/CD log.

**Centralized management:** Một chỗ cập nhật, tất cả app đang dùng secret đó tự nhận giá trị mới.

### Tạo hoặc cập nhật secret

```bash
az containerapp secret set \
    -n ai-api \
    -g rg-aca-demo \
    --secrets embeddings-api-key="REPLACE_WITH_REAL_VALUE"
```

Tên secret (`embeddings-api-key`) là identifier bạn dùng để reference sau này. Giá trị (`REPLACE_WITH_REAL_VALUE`) là string thực sự được lưu — trong production, đây là API key thật.

Lệnh `secret set` có thể tạo mới hoặc update secret đã có — cùng một lệnh cho cả hai trường hợp.

### Key Vault reference (nâng cao)

CLI cũng hỗ trợ reference đến Azure Key Vault thay vì lưu giá trị trực tiếp — khi đó container app fetch secret từ Key Vault lúc runtime. Cách này dùng khi Key Vault là "system of record" cho secret management của tổ chức.

---

## 3. Secret References — Kết nối secret với environment variable

App của bạn thường đọc config qua environment variable. Nhưng bạn không muốn để API key trực tiếp trong env var (sẽ visible trong log, CLI output...). Secret reference giải quyết mâu thuẫn này.

**Cú pháp:** Đặt giá trị của env var là `secretref:<tên-secret>`

```bash
az containerapp update \
    -n ai-api \
    -g rg-aca-demo \
    --set-env-vars EMBEDDINGS_API_KEY=secretref:embeddings-api-key
```

Kết quả: Trong container, biến `EMBEDDINGS_API_KEY` chứa **giá trị thực** của secret `embeddings-api-key` — app code đọc `EMBEDDINGS_API_KEY` bình thường như env var thông thường, không biết giá trị đến từ secret store.

**Luồng hoàn chỉnh:**

```
1. Tạo secret:
   secret name:  embeddings-api-key
   secret value: sk-abc123... (API key thực)

2. Map secret vào env var:
   EMBEDDINGS_API_KEY = secretref:embeddings-api-key

3. Trong container:
   os.environ['EMBEDDINGS_API_KEY'] → "sk-abc123..."
   (app không biết giá trị đến từ secret)
```

---

## 4. YAML — Chuẩn hoá runtime config

Khi app có nhiều env var và secret reference, YAML giúp quản lý tất cả trong một file reviewable.

### YAML fragment cho env vars và secret references

```yaml
# containerapp.yml (fragment)
properties:
  template:
    containers:
    - name: ai-api
      env:
      - name: LOG_LEVEL
        value: info                          # env var thông thường

      - name: EMBEDDINGS_API_KEY
        secretRef: embeddings-api-key        # secret reference
```

**Lưu ý naming:** Trong YAML dùng `secretRef` (camelCase), trong CLI dùng `secretref:` (lowercase với dấu hai chấm). Hai cú pháp khác nhau nhưng cùng tác dụng.

> **Quan trọng:** YAML file lưu trong source control chỉ nên chứa **tên secret** (`embeddings-api-key`), không bao giờ chứa **giá trị secret** (`sk-abc123...`). Giá trị thực được set qua `az containerapp secret set` riêng biệt.

### Apply YAML

```bash
az containerapp update \
    -n ai-api \
    -g rg-aca-demo \
    --yaml ./containerapp.yml
```

---

## So sánh: ACA Secrets vs App Service App Settings

| | ACA Secrets + secretref | App Service App Settings |
|---|---|---|
| Lưu sensitive value | ✅ Secrets store riêng | ✅ Encrypted at rest |
| Tách giá trị khỏi source control | ✅ secretref trong YAML | ⚠️ Cần cẩn thận |
| App đọc qua env var | ✅ secretref map thành env var | ✅ Inject trực tiếp |
| Key Vault integration | ✅ Key Vault reference | ✅ Key Vault reference |
| Rotate không redeploy | ✅ Update secret, app tự nhận | ✅ tương tự |

---

## Best Practices

### Tách config khỏi image

```bash
# Sai — bake config vào image
ENV LOG_LEVEL=production     # Dockerfile

# Đúng — inject lúc runtime
--env-vars LOG_LEVEL=production
```

Cùng image, khác config → deploy được nhiều environment mà không rebuild.

### Luôn dùng secret reference, không để secret value trong env var

```bash
# Sai — API key visible trong CLI history và ACA config
--env-vars EMBEDDINGS_API_KEY=sk-abc123...

# Đúng — dùng secret reference
az containerapp secret set -n ai-api -g rg-aca-demo --secrets embeddings-api-key="sk-abc123..."
az containerapp update -n ai-api -g rg-aca-demo --set-env-vars EMBEDDINGS_API_KEY=secretref:embeddings-api-key
```

### Rotate secret độc lập với image

Khi key bị lộ hoặc hết hạn, chỉ cần:

```bash
az containerapp secret set -n ai-api -g rg-aca-demo \
    --secrets embeddings-api-key="sk-newkey456..."
# Không cần rebuild image, không cần redeploy
```

### Lưu YAML trong source control, không lưu secret value

```yaml
# Git repo — OK
- name: EMBEDDINGS_API_KEY
  secretRef: embeddings-api-key    # chỉ tên, không có giá trị

# Không bao giờ commit cái này
- name: EMBEDDINGS_API_KEY
  value: sk-abc123...              # KHÔNG commit secret value
```

---

## Bản chất bài này là gì?

**Một câu:** ACA tách rõ ràng hai loại config mà Docker và App Service để lẫn lộn — non-sensitive (env var) và sensitive (secret) — với một cơ chế bridge (secretref) để app không cần biết sự khác biệt.

### So sánh với Docker và App Service

| | Docker `docker run` | App Service app settings | ACA |
|---|---|---|---|
| Non-sensitive config | `-e KEY=VALUE` | App settings | `--env-vars KEY=VALUE` |
| Sensitive config | `-e SECRET=abc` (không an toàn) | App settings (encrypted at rest) | `secret set` → tách riêng |
| Bridge sensitive vào env var | Không có | Không tách biệt | `secretref:` syntax |
| Rotate secret không redeploy | Không | ✅ | ✅ |

**Điểm ACA làm rõ hơn App Service:** App Service lưu cả sensitive và non-sensitive trong cùng "app settings". ACA bắt bạn tách ra: sensitive vào secret store, non-sensitive vào env var. Kết quả là codebase an toàn hơn vì secret không bao giờ "lộ" ra trong YAML file.

---

## Checklist ghi nhớ cho AI-200

- [ ] **Environment variables** → non-sensitive config: log level, URL, feature flag, tuning params
- [ ] `--env-vars` dùng với `create` · `--set-env-vars` dùng với `update` (không xoá biến khác)
- [ ] **Secrets** → sensitive config: API key, password, signing key
- [ ] Tạo/update secret: `az containerapp secret set --secrets key="value"`
- [ ] **Secret reference** cú pháp CLI: `--set-env-vars MY_VAR=secretref:<secret-name>`
- [ ] **Secret reference** cú pháp YAML: `secretRef: <secret-name>` (camelCase, không có dấu `:`)
- [ ] App đọc secret qua env var bình thường — không biết giá trị đến từ secret store
- [ ] YAML chỉ lưu **tên secret**, không bao giờ lưu **giá trị secret**
- [ ] Rotate secret không cần rebuild image hay redeploy — chỉ cần `secret set` lại
- [ ] Key Vault reference cũng được hỗ trợ khi Key Vault là system of record

---

*Bài tiếp theo: Bài 4 — Configure image pull authentication for private registries*
