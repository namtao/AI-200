# Bài 3 — Configure Application Settings

> Khoá: AI-200 · ACR — Deploy containers to Azure App Service

---

## Tổng quan

Một trong những nguyên tắc cơ bản của container là **dùng cùng một image cho tất cả environment** (dev, staging, production) — chỉ thay đổi configuration. Bài này giải thích cách App Service inject configuration vào container thông qua environment variable, và các tính năng nâng cao như slot settings và Key Vault references.

```
App Settings (cấu hình)
├── App settings          → environment variable thông thường
├── Connection strings    → environment variable có prefix theo loại DB
├── Bulk editing          → export/import JSON
├── Slot settings         → setting gắn với slot, không swap
└── Key Vault references  → secret được quản lý tập trung
```

---

## 1. App Settings — Environment Variables

App settings là các cặp **name-value** mà App Service inject vào container dưới dạng **environment variable** khi container khởi động. App của bạn đọc các giá trị này bằng cách đọc environment variable — không cần biết nguồn gốc từ đâu.

### Tạo hoặc cập nhật app settings

```bash
az webapp config appsettings set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --settings \
        STORAGE_ACCOUNT_NAME=mystorageaccount \
        LOG_LEVEL=INFO \
        MAX_DOCUMENT_SIZE_MB=50
```

Một lệnh có thể set nhiều setting cùng lúc — liệt kê từng cặp `KEY=VALUE` phân cách bằng dấu cách.

### Đọc app settings trong code

Bên trong container, các setting này là environment variable bình thường:

```python
import os

storage_account = os.environ.get('STORAGE_ACCOUNT_NAME')
log_level = os.environ.get('LOG_LEVEL', 'WARNING')   # 'WARNING' là default nếu không có
max_size = int(os.environ.get('MAX_DOCUMENT_SIZE_MB', 10))
```

### Một số điểm quan trọng cần nhớ

**Encrypted at rest:** Tất cả app settings đều được mã hóa khi lưu trữ. App Service chỉ decrypt khi inject vào container. Điều này áp dụng cho mọi setting, kể cả các giá trị không nhạy cảm.

**Naming rules cho Linux container:** Tên setting chỉ được chứa chữ cái, số, và dấu gạch dưới. Với .NET app dùng nested config (dùng dấu `:` như `ConnectionStrings:DefaultConnection`), trên Linux phải đổi `:` thành `__` (double underscore):

```
.NET config key:        ConnectionStrings:DefaultConnection
Linux env variable:     ConnectionStrings__DefaultConnection
```

---

## 2. Connection Strings — Chuỗi kết nối database

Connection strings là dạng app setting đặc biệt dành cho database connectivity. App Service tự động thêm **prefix** vào tên environment variable dựa trên loại database được khai báo.

### Tạo connection string

```bash
az webapp config connection-string set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --connection-string-type SQLAzure \
    --settings DefaultConnection="Server=myserver.database.windows.net;Database=mydb;..."
```

### Prefix theo loại database

| Connection string type | Environment variable prefix | Ví dụ tên biến |
|---|---|---|
| SQL Server | `SQLCONNSTR_` | `SQLCONNSTR_DefaultConnection` |
| SQL Azure | `SQLAZURECONNSTR_` | `SQLAZURECONNSTR_DefaultConnection` |
| MySQL | `MYSQLCONNSTR_` | `MYSQLCONNSTR_DefaultConnection` |
| PostgreSQL | `POSTGRESQLCONNSTR_` | `POSTGRESQLCONNSTR_DefaultConnection` |
| Custom | `CUSTOMCONNSTR_` | `CUSTOMCONNSTR_DefaultConnection` |

Ví dụ trên dùng `SQLAzure` và tên `DefaultConnection` → environment variable tên là `SQLAZURECONNSTR_DefaultConnection`.

### Khi nào dùng Connection Strings vs App Settings?

Connection strings được thiết kế cho **.NET framework** — vốn có convention đọc connection string từ prefix đặc biệt. Với các ngôn ngữ khác, prefix này chỉ thêm phức tạp mà không có lợi ích gì.

> **Thực tế:** Với Python, Node.js, Go, và hầu hết framework không phải .NET — **dùng App Settings** cho database connection string thay vì Connection Strings. Đơn giản hơn, không cần nhớ prefix.

```python
# Dùng app setting cho Python — đơn giản hơn
db_url = os.environ.get('DATABASE_URL')

# Thay vì phải nhớ prefix POSTGRESQLCONNSTR_
db_url = os.environ.get('POSTGRESQLCONNSTR_DefaultConnection')
```

---

## 3. Bulk Editing — Chỉnh sửa nhiều settings cùng lúc

Khi cần quản lý nhiều setting, thao tác từng cái một bằng CLI rất tốn thời gian. Bulk editing cho phép export ra JSON, chỉnh sửa, rồi import lại.

### Export settings hiện tại ra file

```bash
az webapp config appsettings list \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --output json > settings.json
```

File JSON có cấu trúc:

```json
[
  {
    "name": "STORAGE_ACCOUNT_NAME",
    "value": "mystorageaccount",
    "slotSetting": false
  },
  {
    "name": "LOG_LEVEL",
    "value": "INFO",
    "slotSetting": false
  }
]
```

`slotSetting: false` nghĩa là setting này sẽ swap cùng với app khi swap deployment slot (giải thích ở phần dưới).

### Import settings từ file

```bash
az webapp config appsettings set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --settings @settings.json
```

Dấu `@` trước tên file báo cho CLI biết đọc nội dung từ file thay vì coi đó là string thông thường.

### Qua Azure Portal

Portal có tùy chọn **"Advanced edit"** trong mục Environment Variables — hiển thị tất cả settings dưới dạng JSON để chỉnh sửa trực tiếp, tiện hơn cho người không quen CLI.

---

## 4. Slot Settings — Settings gắn với deployment slot

### Deployment slots là gì?

Deployment slots cho phép chạy **nhiều phiên bản app song song** trong cùng App Service plan, mỗi slot có hostname riêng:

```
production:  myapp.azurewebsites.net
staging:     myapp-staging.azurewebsites.net
```

Bạn deploy phiên bản mới lên `staging`, test xong thì **swap** — staging trở thành production và ngược lại. Không có downtime.

### Vấn đề khi swap

Khi swap, toàn bộ app settings cũng swap theo. Điều này gây ra vấn đề: production database URL có thể bị swap sang staging, và staging database URL bị swap vào production — đây là thảm họa.

### Giải pháp: Slot settings

Slot setting là setting **gắn với slot, không swap** khi bạn swap hai slot. Nó "ở lại" với slot dù code có swap qua lại.

```bash
az webapp config appsettings set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --slot staging \
    --slot-settings \
        ENVIRONMENT=staging \
        API_ENDPOINT=https://api-staging.example.com
```

`--slot staging` chỉ định đang cấu hình cho staging slot. `--slot-settings` (thay vì `--settings`) đánh dấu các setting này là slot-specific.

**Khi swap xảy ra:**

```
Trước swap:
  production slot: ENVIRONMENT=production, DATABASE_URL=prod-db
  staging slot:    ENVIRONMENT=staging,    DATABASE_URL=staging-db   ← slot settings

Sau swap:
  production slot: ENVIRONMENT=staging(!)  DATABASE_URL=prod-db(!)   ← SAI nếu không dùng slot settings
  staging slot:    ENVIRONMENT=production  DATABASE_URL=staging-db(!)← SAI

Với slot settings (ENVIRONMENT là slot setting):
  production slot: ENVIRONMENT=production ← GIỮ NGUYÊN  DATABASE_URL=staging-db (đã swap)
  staging slot:    ENVIRONMENT=staging    ← GIỮ NGUYÊN  DATABASE_URL=prod-db (đã swap)
```

### Settings nên đánh dấu là slot settings

- **Environment identifier:** `ENVIRONMENT=production` — không bao giờ nên swap
- **Environment-specific endpoints:** API URL, database URL khác nhau giữa môi trường
- **Feature flags:** Tính năng chỉ bật ở staging để test
- **Logging level:** Verbose logging ở staging không nên ảnh hưởng production

### Kiểm tra settings nào đang là slot settings

```bash
az webapp config appsettings list \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --query "[?slotSetting==\`true\`].name"
```

---

## 5. Key Vault References — Quản lý secret tập trung

### Vấn đề với secret trong app settings

Dù app settings được encrypt at rest, lưu secret (API key, password, certificate) trực tiếp trong app settings vẫn có một số hạn chế:

- Không có audit trail — không biết ai đã đọc secret
- Khó rotate — phải cập nhật nhiều nơi cùng lúc
- Secret bị phân tán ở nhiều App Service, khó quản lý tập trung

### Giải pháp: Azure Key Vault + Key Vault References

Lưu secret trong Azure Key Vault, rồi trong app settings chỉ chứa **reference** trỏ đến Key Vault:

```bash
az webapp config appsettings set \
    --resource-group myResourceGroup \
    --name myDocumentProcessor \
    --settings \
        API_KEY="@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/api-key)"
```

App Service tự động resolve reference này — khi inject vào container, biến `API_KEY` sẽ chứa **giá trị thực** của secret, không phải URL Key Vault. App code đọc `API_KEY` bình thường, không cần biết giá trị đến từ Key Vault.

### Yêu cầu để dùng Key Vault references

1. **Managed identity** phải được bật trên web app
2. Managed identity đó phải được cấp quyền **read secrets** từ Key Vault (role `Key Vault Secrets User`)
3. Dùng đúng cú pháp `@Microsoft.KeyVault(...)` trong giá trị setting

### Secret rotation tự động

Khi không chỉ định version trong reference URL, App Service luôn lấy **latest version** của secret:

```
https://myvault.vault.azure.net/secrets/api-key        ← luôn lấy latest
https://myvault.vault.azure.net/secrets/api-key/abc123 ← cố định version cụ thể
```

Khi secret được rotate (cập nhật version mới trong Key Vault), App Service tự động refresh giá trị trong vòng **24 giờ**. Nếu cần refresh ngay, trigger một app restart — restart sẽ force refetch tất cả Key Vault references.

> **Lưu ý cho thi:** Key Vault references cần **managed identity** trên web app. Đây là một trong những use case phổ biến nhất của managed identity.

---

## 6. Verify Configuration — Kiểm tra settings đã được áp dụng

Sau khi cấu hình, cần xác nhận app nhận đúng giá trị. Có hai cách:

### Dùng SCM (Kudu) site

Mỗi App Service có một site quản trị đi kèm gọi là **Kudu** hoặc SCM site:

```
https://<app-name>.scm.azurewebsites.net
```

Vào **Environment** (hoặc truy cập trực tiếp `https://<app-name>.scm.azurewebsites.net/Env`) để xem toàn bộ environment variable mà App Service inject vào container — bao gồm app settings của bạn lẫn các system variable của Azure.

Đây là cách nhanh nhất để debug "tại sao app không đọc được setting X".

### Dùng diagnostic endpoint trong app

Thêm một endpoint trả về non-sensitive config để verify:

```python
@app.route('/config-check')
def config_check():
    return {
        'log_level': os.environ.get('LOG_LEVEL'),
        'storage_account': os.environ.get('STORAGE_ACCOUNT_NAME'),
        'max_doc_size': os.environ.get('MAX_DOCUMENT_SIZE_MB')
        # Không trả về secret/password ở đây
    }, 200
```

Chỉ expose endpoint này trong development/staging — không expose trong production.

---

## Tóm tắt — Khi nào dùng cái nào?

| Tình huống | Giải pháp |
|---|---|
| Config thông thường (URL, tên account, log level...) | App settings |
| Database connection string cho .NET | Connection strings |
| Database connection string cho Python/Node.js | App settings (đơn giản hơn) |
| Setting phải ở lại khi swap slot (env name, env-specific URL) | Slot settings |
| API key, password, certificate cần audit trail và rotation | Key Vault references |
| Cần chỉnh nhiều settings cùng lúc | Bulk edit qua JSON |

---

## Bản chất bài này là gì?

**Một câu:** Bài này hiện thực hóa nguyên tắc "build once, run anywhere" — cùng một image, thay đổi hành vi qua environment variable thay vì rebuild.

Với Docker thuần, bạn inject config qua `docker run -e KEY=VALUE` hoặc `--env-file .env`. App Service làm điều tương tự, nhưng "env file" được quản lý tập trung qua Portal/CLI, mã hóa tự động, và có thêm 3 tính năng vượt trội mà Docker không có sẵn.

### So sánh với Docker và các giải pháp khác

| Tính năng | Docker thuần | Docker Compose | Kubernetes | App Service |
|---|---|---|---|---|
| Inject config | `docker run -e KEY=VALUE` | `environment:` trong compose.yml | ConfigMap | App settings |
| Quản lý nhiều config | `.env` file thủ công | `.env` + profiles | ConfigMap manifest | Bulk edit JSON |
| Secret | `.env` file (không an toàn) | Docker secrets | Kubernetes Secret | Key Vault references |
| Config theo môi trường | Nhiều `.env` file | Compose profiles / override file | Namespace + Helm values | Slot settings |
| Audit trail secret | Không có | Không có | Hạn chế | Azure Key Vault full audit |

**Insight quan trọng:** App settings KHÔNG phải file `.env` baked trong image. Chúng được inject lúc runtime khi container start — image không chứa config, config đến từ ngoài. Đây là lý do bạn có thể dùng cùng image `docprocessor:v1` cho cả staging lẫn production với config khác nhau.

**3 điểm khác biệt nổi bật so với Docker:**

1. **Slot settings — không có tương đương trong Docker:** Docker không có khái niệm "deployment slot swap". Slot settings giải quyết vấn đề: khi swap staging↔production, config nào nên swap theo code và config nào nên ở lại với environment. Gần nhất là Docker Compose profiles, nhưng không tự động xử lý khi swap.

2. **Key Vault references — secret không bao giờ chạm App Service:** Với Docker thuần, secret ở `.env` file → tồn tại trên disk. Với App Service + Key Vault references, App Service chỉ lưu *URL trỏ đến secret*, giá trị thực nằm trong Key Vault và chỉ được fetch lúc container start. Tương đương Kubernetes External Secrets hoặc HashiCorp Vault agent injection.

3. **Connection strings với prefix tự động — legacy .NET feature:** Bài học thực tế là *không cần dùng* nếu bạn làm Python/Node.js. Đây là legacy behavior từ thời .NET Framework, không phải best practice mới.

---

## Checklist ghi nhớ cho AI-200

- [ ] App settings được inject vào container dưới dạng **environment variable**
- [ ] Tất cả app settings được **encrypted at rest**
- [ ] Tên setting chỉ dùng chữ, số, dấu gạch dưới. Nested key dùng `__` (double underscore) thay vì `:` trên Linux
- [ ] Connection strings tự động thêm **prefix** theo loại DB (`SQLAZURECONNSTR_`, `POSTGRESQLCONNSTR_`,...)
- [ ] Với Python/Node.js: dùng **app settings** cho DB connection, không dùng connection strings
- [ ] Bulk edit: export bằng `appsettings list --output json`, import bằng `appsettings set --settings @file.json`
- [ ] **Slot settings** gắn với slot, không swap khi bạn swap deployment slot — dùng cho env identifier, env-specific URL
- [ ] `--slot-settings` (thay vì `--settings`) để đánh dấu là slot setting
- [ ] **Key Vault references** cú pháp: `@Microsoft.KeyVault(SecretUri=https://...)`
- [ ] Key Vault references yêu cầu **managed identity** có quyền đọc secret từ Key Vault
- [ ] Secret không chỉ định version → luôn lấy **latest version**, auto-refresh trong 24 giờ khi secret được rotate
- [ ] Verify settings qua **Kudu/SCM site**: `https://<app-name>.scm.azurewebsites.net/Env`

---

[← Bài 2](./acr-m2-bai2-configure-container-runtime.md) · [🏠 Mục lục](../README.md) · [Bài 4 →](./acr-m2-bai4-observe-troubleshoot.md)
