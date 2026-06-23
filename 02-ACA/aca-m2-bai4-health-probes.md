# Bài 4 — Configure Health Probes and Troubleshoot Failures

> Khoá: AI-200 · ACA — Manage containers in Azure Container Apps

---

## Tại sao health probe quan trọng với AI service?

Health probe bảo vệ user trong quá trình rollout bằng cách ngăn traffic đến replica chưa sẵn sàng. Với AI API, điều này đặc biệt quan trọng vì startup thường bao gồm:

- **Load model vào memory** — có thể tốn 30-120 giây với model lớn
- **Warm up cache** — preload frequent queries hoặc embeddings
- **Establish connections** — kết nối đến database, vector store, model endpoint

Nếu không có readiness probe, ACA có thể route traffic đến replica **đang load model** — request sẽ fail hoặc có latency rất cao. Probe ngăn điều này.

> **Lưu ý:** Probe schema trong YAML có thể thay đổi. Validate với documentation hiện tại trước khi dùng trong production.

---

## Readiness vs Liveness — Hai câu hỏi khác nhau

Đây là điểm quan trọng nhất của bài — hai loại probe có **mục đích khác nhau hoàn toàn**:

### Readiness Probe — "Replica này có thể nhận traffic ngay bây giờ không?"

- Nếu **fail**: Platform **không route traffic** đến replica này — replica vẫn chạy nhưng bị tạm thời loại khỏi load balancer
- Nếu **pass**: Traffic được route bình thường
- **Dùng để:** Guard cho model loading và dependency readiness

**Ví dụ với AI service:**

```
Replica start → bắt đầu load BERT model (30 giây)
     ↓
Readiness probe gọi /health/ready → return 503 (model chưa load xong)
     ↓
ACA: không route traffic đến replica này
     ↓
Model load xong → /health/ready return 200
     ↓
ACA: bắt đầu route traffic
```

User không bao giờ nhận request fail do model chưa load — readiness probe bảo vệ họ.

### Liveness Probe — "Process này còn đủ khỏe để tiếp tục chạy không?"

- Nếu **fail** liên tục: Platform **restart replica**
- Nếu **pass**: Không làm gì
- **Dùng để:** Safety valve cho deadlock, memory leak, hoặc process bị stuck

**Ví dụ với AI service:**

```
Replica đang chạy → xảy ra memory leak sau nhiều giờ
     ↓
Liveness probe gọi /health/live → timeout (process quá chậm)
     ↓
ACA: restart replica (clear memory leak)
```

### Tóm tắt sự khác biệt

| | Readiness | Liveness |
|---|---|---|
| Câu hỏi | Sẵn sàng nhận traffic? | Còn sống và healthy? |
| Khi fail | Ngừng route traffic | Restart replica |
| Dùng cho AI | Guard model loading | Safety valve deadlock/memory |
| Nên aggressive? | Không — cho đủ thời gian load | Không — restart thường = cold start |

---

## Cấu hình probes trong YAML

```yaml
# containerapp.yml (fragment)
properties:
  template:
    containers:
    - name: api
      image: <registry>/<repo>@sha256:<digest>
      probes:
      - type: Readiness
        httpGet:
          path: /health/ready
          port: 8080
        initialDelaySeconds: 20     # Chờ 20s sau khi container start mới bắt đầu probe
        periodSeconds: 10           # Probe mỗi 10 giây
        timeoutSeconds: 2           # Timeout nếu không response trong 2 giây
        failureThreshold: 3         # Fail 3 lần liên tiếp → loại khỏi load balancer

      - type: Liveness
        httpGet:
          path: /health/live
          port: 8080
        initialDelaySeconds: 60     # Chờ 60s (đủ thời gian load model)
        periodSeconds: 10
        timeoutSeconds: 2
        failureThreshold: 3         # Fail 3 lần liên tiếp → restart
```

### Giải thích từng parameter

**`initialDelaySeconds`** — Thời gian chờ sau khi container start trước khi bắt đầu probe.

Đây là parameter quan trọng nhất với AI service. Nếu quá ngắn:
- Readiness probe gọi vào lúc model đang load → fail → ACA không route traffic → OK
- Liveness probe gọi vào lúc model đang load → fail → ACA restart container → vòng lặp vô tận

Rule of thumb: `initialDelaySeconds` của liveness nên **lớn hơn** thời gian load model. Nếu model mất 45 giây để load, set ít nhất `initialDelaySeconds: 60`.

**`periodSeconds`** — Tần suất probe. Mỗi N giây probe một lần.

**`timeoutSeconds`** — Bao lâu chờ response trước khi tính là fail. Health endpoint phải **nhẹ và nhanh** — không thực hiện operation nặng.

**`failureThreshold`** — Fail bao nhiêu lần liên tiếp mới trigger action (deactivate khỏi LB hoặc restart). Tăng threshold để chấp nhận transient failure mà không restart ngay.

### Hai endpoint khác nhau

Readiness và liveness nên có **endpoint riêng** vì logic check khác nhau:

```python
@app.route('/health/ready')
def readiness():
    """Chỉ return 200 khi model đã load xong và dependencies healthy"""
    if not model_loaded:
        return {'status': 'loading'}, 503
    try:
        db.ping()
        return {'status': 'ready'}, 200
    except Exception:
        return {'status': 'dependency_unavailable'}, 503

@app.route('/health/live')
def liveness():
    """Chỉ return 200 khi process còn alive và không bị stuck"""
    # Kiểm tra đơn giản — process có respond không?
    # Không check external dependency vì nếu DB down, không nên restart app
    return {'status': 'alive'}, 200
```

> **Lưu ý quan trọng:** Liveness endpoint **không nên check external dependency** như database hay model provider. Nếu DB down mà liveness trả 503, ACA sẽ restart container liên tục — không giải quyết được DB down, chỉ tạo thêm cold start.

---

## Common probe failure causes — Kiểm tra đây trước

Probe failure thường là **configuration issue** chứ không phải code issue. Trước khi debug sâu, validate 4 nguyên nhân phổ biến này:

**1. Probe trỏ sai port**

```yaml
# Sai — container listen 8000, probe gọi 8080
httpGet:
  port: 8080    # ← sai

# Đúng
httpGet:
  port: 8000
```

**2. Probe trỏ sai path**

```yaml
# Sai — app expose /health/ready nhưng probe gọi /health
httpGet:
  path: /health    # ← sai

# Đúng
httpGet:
  path: /health/ready
```

**3. Timeout quá ngắn cho model loading**

```yaml
# Sai — model load 45s nhưng initialDelay chỉ 10s
initialDelaySeconds: 10    # ← probe bắt đầu khi model đang load → liveness restart loop

# Đúng
initialDelaySeconds: 60    # Đủ thời gian cho model load
```

**4. Startup cần external dependency và dependency đó chậm hoặc unavailable**

Readiness gọi `/health/ready` → endpoint check DB → DB đang warm up → timeout → probe fail.

Giải pháp: Implement retry logic trong health endpoint, hoặc tách "DB connected" ra khỏi initial readiness check.

---

## Troubleshoot probe failures trong rollout

### Bước 1: Kiểm tra revision health state

```bash
az containerapp revision list \
    --name ai-api \
    --resource-group rg-aca-demo \
    --query "[].{name:name,active:properties.active,health:properties.healthState}" \
    -o table
```

### Bước 2: Stream log để xem probe được gọi và response thế nào

```bash
az containerapp logs show \
    --name ai-api \
    --resource-group rg-aca-demo \
    --follow --tail 50
```

Trong log, tìm:
- HTTP request đến `/health/ready` và `/health/live` — đây là probe request
- Response status code — 200 = pass, khác = fail
- Startup log — model load đến đâu rồi?

### Bước 3: Điều chỉnh probe settings nếu cần

Nếu liveness fail ngay sau startup → tăng `initialDelaySeconds` hoặc `failureThreshold`.

Nếu readiness pass sớm nhưng app chưa thực sự ready → health endpoint implementation cần check kỹ hơn.

---

## Checklist ghi nhớ cho AI-200

- [ ] **Readiness** fail → ngừng route traffic (replica vẫn chạy)
- [ ] **Liveness** fail liên tục → restart replica
- [ ] Với AI service: readiness guard model loading · liveness safety valve cho deadlock/memory
- [ ] **Không dùng liveness quá aggressive** — restart thường = cold start = model load lại = latency spike
- [ ] `initialDelaySeconds` của liveness phải **lớn hơn** thời gian load model
- [ ] Readiness và liveness nên có **endpoint riêng biệt** với logic check khác nhau
- [ ] **Liveness endpoint không check external dependency** — DB down không nên trigger restart
- [ ] 4 nguyên nhân phổ biến: sai port, sai path, timeout quá ngắn, dependency chậm
- [ ] Troubleshoot: `revision list` health state + `logs show` để xem probe response
- [ ] Health endpoint phải **nhẹ và trả về nhanh** — không làm operation nặng

---

*Bài tiếp theo: Bài 5 — Optimize container resources and scaling*
