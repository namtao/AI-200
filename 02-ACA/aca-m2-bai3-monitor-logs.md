# Bài 3 — Monitor Logs and Troubleshoot Issues

> Khoá: AI-200 · ACA — Manage containers in Azure Container Apps

---

## Vai trò của log trong managed container platform

Với managed platform như ACA, bạn **không có shell access** vào container đang chạy (không có SSH như App Service). Log là **tín hiệu diagnostic chính** — thứ duy nhất cho bạn biết chuyện gì đang xảy ra bên trong.

Với AI solution, log giúp:
- Correlate request với model version đang phục vụ
- Giải thích latency spike (dependency nào chậm?)
- Identify failure pattern (lỗi chỉ xảy ra với input loại nào?)

> **Lưu ý:** Log access phụ thuộc vào cách environment được cấu hình. Đảm bảo environment đã connect với **Log Analytics** hoặc dùng **log streaming** cho real-time investigation.

---

## Log Streaming — Real-time diagnosis

Log streaming phù hợp khi bạn có thể **reproduce issue ngay lúc đó**:
- Request trigger exception
- Revision mới đang crash loop lúc startup
- Muốn confirm config change có hiệu lực không

```bash
az containerapp logs show \
    --name <app-name> \
    --resource-group <resource-group> \
    --follow
```

`--follow` stream log liên tục — bạn thấy event ngay khi xảy ra, không cần query sau.

**Dùng hiệu quả hơn khi kết hợp với structured logging:** Nếu app log kèm request ID và revision ID, bạn có thể filter ngay trong output thay vì phải đọc toàn bộ stream.

---

## Thiết kế log tốt cho AI service

AI service có đặc điểm khác với web app thông thường: **behavior thay đổi theo input**. Request A có thể thành công, request B với cấu trúc tương tự nhưng content khác lại fail. Log không đủ context sẽ không giúp được gì.

### Các field nên có trong application log

| Field | Tại sao cần |
|---|---|
| **Request identifier / Correlation ID** | Trace request qua nhiều service — từ API → text extractor → embedding service |
| **Revision hoặc build identifier** | Biết đang chạy version nào — match với image tag/digest hoặc version string baked vào image |
| **Model version** | Model deployment name hoặc version dùng cho request — quan trọng khi A/B test model |
| **Latency breakdown** | Total time + timing của từng dependency — xác định bottleneck |

### Cân bằng giữa detail và privacy

AI service thường xử lý tài liệu, prompt, hoặc dữ liệu người dùng — **không log raw content**.

```python
# Không làm thế này — log raw document content
logger.info(f"Processing document: {document_content}")

# Làm thế này — log metadata và identifier
logger.info(f"Processing document: id={doc_id}, size_kb={len(document_content)//1024}, "
            f"revision={os.environ.get('REVISION_ID')}, model={model_version}, "
            f"correlation_id={correlation_id}")
```

---

## Quy trình troubleshoot revision failure

Khi có incident, workflow sau giúp narrow scope nhanh:

**Bước 1 — Xác định revision nào đang lỗi**

```bash
az containerapp revision list \
    --name ai-api \
    --resource-group rg-aca-demo \
    --query "[].{name:name,active:properties.active,health:properties.healthState}" \
    -o table
```

**Bước 2 — Stream log trong khi reproduce hoặc revision đang start**

```bash
az containerapp logs show \
    --name ai-api \
    --resource-group rg-aca-demo \
    --follow --tail 50
```

**Bước 3 — So sánh config giữa revision hoạt động và revision lỗi**

```bash
# Xem detail revision lỗi
az containerapp revision show \
    --name ai-api \
    --resource-group rg-aca-demo \
    --revision <failing-revision-name>

# So sánh với revision đang hoạt động
az containerapp revision show \
    --name ai-api \
    --resource-group rg-aca-demo \
    --revision <working-revision-name>
```

Tìm điểm khác biệt: image reference khác không? Env var nào bị thiếu? Resource limit có thay đổi?

**Bước 4 — Apply fix và validate revision mới**

Deploy revision mới với fix, rồi theo dõi:
```bash
az containerapp revision list \
    --name ai-api \
    --resource-group rg-aca-demo \
    -o table
```

Confirm revision mới có `healthState: Healthy` trước khi kết thúc incident.

---

## Các loại revision failure và cách nhận biết qua log

| Failure type | Dấu hiệu trong log |
|---|---|
| **Startup failure** | App exit ngay, log dừng đột ngột sau vài dòng startup |
| **Missing env var** | Error message "KeyError", "undefined", "required environment variable not set" |
| **Probe failure** | App start OK nhưng health check endpoint trả lỗi hoặc timeout |
| **Dependency connection** | Timeout, connection refused khi kết nối database/cache/model API |
| **OOM kill** | Log dừng đột ngột, không có error message (platform kill process) |

---

## Log Analytics — Long-term query

Với Log Analytics (khi environment được cấu hình), dùng KQL để query log lịch sử:

```kusto
// Tìm lỗi trong 1 giờ qua
ContainerAppConsoleLogs_CL
| where TimeGenerated > ago(1h)
| where Log_s contains "ERROR"
| project TimeGenerated, RevisionName_s, Log_s
| order by TimeGenerated desc
```

```kusto
// Xem latency pattern theo revision
ContainerAppConsoleLogs_CL
| where Log_s contains "latency_ms"
| parse Log_s with * "latency_ms=" latency:int *
| summarize avg_latency=avg(latency) by RevisionName_s, bin(TimeGenerated, 5m)
```

---

## Bản chất bài này là gì?

**Một câu:** ACA không có shell access vào container → log là cửa sổ duy nhất vào runtime; structured logging không còn là "best practice" mà là yêu cầu bắt buộc.

### So sánh toolchain

| | Docker | App Service | ACA |
|---|---|---|---|
| Xem log container | `docker logs` | `az webapp log tail` | `az containerapp logs show` |
| Shell vào container | `docker exec -it bash` (luôn có) | SSH (cần cấu hình) | ❌ Không có |
| Log dài hạn | Docker logging driver (ELK, Splunk) | Log Analytics | Log Analytics + KQL |
| Trace qua nhiều service | Phải tự setup | Không có built-in | Tất cả app trong environment đổ về cùng workspace |

**Điểm khác biệt cốt lõi với App Service:** App Service có Kudu = can browse env var, file system, SSH. ACA không có. Log là thứ duy nhất bạn có → thiết kế log tốt từ đầu quan trọng hơn nhiều so với App Service.

**Hệ quả thực tế:** App log cần include correlation ID + revision ID + model version. Không có những field này, khi incident xảy ra bạn không có cách narrow scope.

---

## Checklist ghi nhớ cho AI-200

- [ ] Log là **tín hiệu diagnostic chính** trong ACA — không có shell access vào container
- [ ] Log access cần environment connect với **Log Analytics** hoặc dùng log streaming
- [ ] Real-time log: `az containerapp logs show --follow`
- [ ] 4 field quan trọng trong AI service log: **correlation ID**, **revision/build ID**, **model version**, **latency breakdown**
- [ ] **Không log raw document/prompt content** — log metadata và identifier thay thế
- [ ] Quy trình troubleshoot: xác định revision lỗi → stream log → so sánh config → fix → validate
- [ ] `revision show` để so sánh config chi tiết giữa revision hoạt động và revision lỗi
- [ ] OOM kill không có error message trong log — process bị kill bởi platform
- [ ] Log Analytics + KQL cho long-term analysis và pattern detection

---

[← Bài 2](./aca-m2-bai2-container-lifecycle.md) · [🏠 Mục lục](../README.md) · [Bài 4 →](./aca-m2-bai4-health-probes.md)
