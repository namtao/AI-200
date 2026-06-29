# Bài 2 — Set Up the Local Development Environment for Functions

> Khoá: AI-200 · Azure Functions — Build serverless AI backends with Azure Functions

---

## Tools Cần Thiết

- **Azure Functions Core Tools v4** — local runtime
- **VS Code + Azure Functions extension**
- Python 3.9+

```bash
# macOS
brew tap azure/functions && brew install azure-functions-core-tools@4

# Windows
winget install Microsoft.Azure.FunctionsCoreTools

# Verify
func --version
```

---

## Project Structure (Python v2)

```
my-ai-backend/
├── function_app.py         # Entry point — tất cả functions + decorators
├── host.json               # Global runtime config (timeout, logging, concurrency)
├── local.settings.json     # Local app settings + connection strings (KHÔNG commit)
├── requirements.txt        # Python dependencies
├── .funcignore             # Exclude từ deployment package
└── .vscode/
    ├── launch.json         # Debugger config — attach đến Core Tools
    ├── tasks.json          # Build + host start tasks
    └── extensions.json     # Recommend Azure Functions extension
```

**Python v2:** Single `function_app.py` với decorators. Dùng **blueprints** khi functions nhiều (split thành modules, register vào main app).

**`.vscode/` nên commit** — team chia sẻ consistent dev experience. **`local.settings.json` KHÔNG commit** — chứa secrets.

---

## Azurite — Local Storage Emulator

Azure Functions runtime cần storage connection (`AzureWebJobsStorage`) để coordinate triggers, track state, store function keys. **Non-HTTP triggers** (queue, timer) không work nếu thiếu.

```json
// local.settings.json
{
    "IsEncrypted": false,
    "Values": {
        "AzureWebJobsStorage": "UseDevelopmentStorage=true",
        "FUNCTIONS_WORKER_RUNTIME": "python"
    }
}
```

`UseDevelopmentStorage=true` → Azurite. Chỉ có hiệu lực khi local — không effect trên Azure.

**Start Azurite:** VS Code extension (auto) hoặc `npm install -g azurite && azurite --silent`.

---

## Local Services cho AI Backend

| Service | Local emulator/alternative |
|---|---|
| Azure Cosmos DB | Cosmos DB emulator (Windows/Linux/macOS via Docker) |
| PostgreSQL | `docker run -e POSTGRES_PASSWORD=mysecret -p 5432:5432 postgres` |
| Redis | `docker run -p 6379:6379 redis` |

Configure connection strings trong `local.settings.json` → swap với production khi deploy.

---

## Start và Debug

```bash
# Start từ terminal
func start

# Hoặc F5 trong VS Code (attach debugger)
# Endpoint: http://localhost:7071
```

### Test HTTP triggers

```bash
curl -X POST http://localhost:7071/api/classify \
    -H "Content-Type: application/json" \
    -d '{"document_url": "https://storage.example.com/docs/sample.pdf"}'
```

### Test Service Bus triggers

Gửi message qua Azure portal Service Bus Explorer hoặc Azure CLI.

---

## Sync Settings

```bash
# Download settings từ deployed app
func azure functionapp fetch-app-settings <app-name>

# Encrypt local settings (bảo vệ secrets trên dev machine)
func settings encrypt
```

**Cẩn thận khi push settings lên Azure** — không overwrite production settings bằng local dev values. Production settings quản lý qua Azure CLI hoặc IaC templates.

---

## Bản chất bài này là gì?

**Một câu:** Local dev Azure Functions = Core Tools v4 + Azurite (local storage emulator, bắt buộc cho non-HTTP triggers) + local.settings.json (KHÔNG commit) — setup đúng để trigger hoạt động trước khi deploy.

### So sánh với AWS SAM Local / Google Functions Framework / Serverless Framework

| | Azure Functions Core Tools | AWS SAM Local | Google Functions Framework | Serverless Framework |
|---|---|---|---|---|
| Local runtime | Đầy đủ (HTTP + triggers) | Đầy đủ | HTTP only | Wrap cloud APIs |
| Storage emulator | Azurite (bắt buộc) | LocalStack | Không | LocalStack |
| Config file | local.settings.json | .env / samconfig | .env | serverless.yml |
| Debug support | VS Code attach | VS Code / IntelliJ | VS Code | Limited |
| Non-HTTP triggers local | Có (cần Azurite) | Có | Không | Không |
| Deploy command | `func azure functionapp publish` | `sam deploy` | `gcloud functions deploy` | `serverless deploy` |
| Project init | `func init` | `sam init` | `functions-framework` | `serverless create` |

**Tại sao Azurite bắt buộc:** Functions runtime dùng Azure Storage để coordinate trigger processing và track state ngay cả khi trigger là Service Bus hay Timer — không phải chỉ cho Blob/Queue triggers. Non-HTTP trigger không fire nếu thiếu.

**Exam trap:** `local.settings.json` KHÔNG commit vào git (chứa secrets). `.vscode/` nên commit (team chia sẻ debug config). Hai quy tắc này hay bị nhầm trong exam.

---

## Checklist ghi nhớ cho AI-200

- [ ] `func --version` verify Core Tools installation
- [ ] Python v2: single `function_app.py` với decorators · Blueprints cho large projects
- [ ] `host.json` = global config (timeout, logging, concurrency)
- [ ] `local.settings.json` = local app settings — **KHÔNG commit**
- [ ] `.vscode/` nên commit — team sharing
- [ ] `AzureWebJobsStorage = "UseDevelopmentStorage=true"` → Azurite
- [ ] Non-HTTP triggers cần valid `AzureWebJobsStorage` để hoạt động local
- [ ] `func start` hoặc F5 → endpoint tại `http://localhost:7071`
- [ ] `func azure functionapp fetch-app-settings` → download settings từ Azure

---

*Bài tiếp theo: Bài 3 — Create triggers and bindings for AI integration patterns*
