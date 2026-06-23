# Module Assessment — Manage Containers in Azure Container Apps

> Khoá: AI-200 · ACA — Manage containers in Azure Container Apps

---

## Câu 1

**You deploy a new image to a production container app. Which approach provides the best traceability and reduces the risk of deploying the wrong artifact?**

- Reference the image by digest (for example, `myregistry.azurecr.io/app@sha256:<digest>`) when you update the container app. ✅
- Reference the image by the `latest` tag to ensure the platform always pulls the most recent build.
- Rebuild the image locally on each environment to ensure the image matches the environment.

**Đáp án: Reference the image by digest**

Digest là **immutable identifier** — một khi image được push, `sha256:<digest>` đó gắn với image đó mãi mãi. Hai lợi ích trực tiếp:

- **Traceability:** Bạn có thể prove chính xác artifact nào đang chạy trong production. Khi có incident, digest trong revision config cho bạn biết ngay đó là build nào.
- **Risk reduction:** Không thể vô tình deploy sai image vì digest chỉ trỏ đúng một image duy nhất — không ai có thể push image khác vào cùng digest.

```bash
az containerapp update \
    --name ai-api \
    --resource-group rg-aca-demo \
    --image myregistry.azurecr.io/ai-api@sha256:a1b2c3d4e5f6...
```

Tại sao các đáp án còn lại sai:
- **`latest` tag:** Tag là mutable — ai đó có thể push image khác đè lên `latest`. Platform pull `latest` và nhận image khác với kỳ vọng, không có cách nào biết được. Đây chính xác là vấn đề mà digest giải quyết.
- **Rebuild locally:** Rebuild từng environment phá vỡ nguyên tắc cơ bản của container — "build once, run anywhere". Image rebuilt trên máy khác nhau có thể có dependency khác nhau dù cùng Dockerfile. Không có audit trail giữa environments.

---

## Câu 2

**A new revision starts but shouldn't receive traffic while you investigate. Which action removes the revision from traffic without deleting it?**

- Run `az containerapp revision deactivate` for the revision. ✅
- Run `az containerapp revision delete` for the revision.
- Run `az containerapp stop` for the container app.

**Đáp án: `az containerapp revision deactivate`**

```bash
az containerapp revision deactivate \
    --name ai-api \
    --resource-group rg-aca-demo \
    --revision <revision-name>
```

Deactivate làm đúng những gì câu hỏi yêu cầu: revision **ngừng nhận traffic** nhưng **không bị xoá**. Revision vẫn còn với đầy đủ config, log, và artifact — bạn có thể tiếp tục điều tra và reactivate lại khi cần.

Tại sao các đáp án còn lại sai:
- **`revision delete`:** Xoá hoàn toàn revision — mất evidence (config, log history), mất rollback option. Delete là irreversible và không nên là bước đầu trong incident response. Nguyên tắc: deactivate trước, investigate, delete cuối cùng khi chắc chắn không cần nữa.
- **`containerapp stop`:** Stop toàn bộ app — ảnh hưởng tất cả revision, kể cả revision cũ đang healthy đang phục vụ user. Đây là action quá broad khi vấn đề chỉ ở một revision cụ thể. User không nhận được service trong thời gian app bị stop.

---

## Câu 3

**A revision fails readiness checks immediately after deployment. Which issue is the most common root cause you should validate first?**

- The readiness probe targets the wrong port or path for the container's HTTP server. ✅
- The container image is too small to include all required libraries.
- The container app environment can't route traffic to the public internet.

**Đáp án: Wrong port or path**

Probe failure thường là **configuration issue** chứ không phải code issue. Khi readiness fail ngay sau deployment, nguyên nhân phổ biến nhất là misconfiguration — probe đang gọi vào địa chỉ không đúng:

```yaml
# Sai — container listen 8000 nhưng probe gọi 8080
probes:
- type: Readiness
  httpGet:
    path: /health/ready
    port: 8080         # ← sai port

# Sai — app expose /health/ready nhưng probe gọi /health
probes:
- type: Readiness
  httpGet:
    path: /health      # ← sai path
    port: 8000
```

Validate bằng cách kiểm tra port trong startup command và path trong health endpoint implementation, rồi đối chiếu với probe config.

Tại sao các đáp án còn lại sai:
- **Image too small:** Nếu image thiếu library, app sẽ crash khi startup với import error — điều này thể hiện trong **container log** như exception, không phải chỉ fail readiness probe im lặng. Hơn nữa, image thường đã được test trước khi push lên registry.
- **Environment can't route to internet:** Readiness probe là **internal check** — ACA platform gọi vào container bên trong environment, không đi qua internet. Vấn đề internet routing không ảnh hưởng đến readiness probe của chính container đó.

---

## Câu 4

**You suspect only one revision has a runtime exception after an update. What is the most effective first step to confirm the problem?**

- Stream the container app logs and filter by revision while reproducing the request. ✅
- Delete older revisions to reduce noise in troubleshooting output.
- Increase CPU and memory allocations before investigating.

**Đáp án: Stream logs and filter by revision**

```bash
az containerapp logs show \
    --name ai-api \
    --resource-group rg-aca-demo \
    --follow --tail 50
```

Log stream là **fastest diagnostic signal** — bạn thấy exception, stack trace, và error message ngay khi chúng xảy ra. Với scaled-out app, log stream bao gồm output từ tất cả instance với revision identifier — bạn có thể filter để chỉ xem log từ revision nghi ngờ.

Đây cũng là approach **non-destructive** — bạn chỉ đọc, không thay đổi gì, preserve toàn bộ evidence.

Tại sao các đáp án còn lại sai:
- **Delete older revisions:** Đây là điều **không nên làm** khi troubleshoot. Revision cũ là reference point — bạn cần compare config và behavior giữa revision hoạt động và revision lỗi để tìm điểm khác biệt. Xoá revision cũ = mất khả năng compare = khó diagnose hơn.
- **Increase CPU/memory:** Tăng resource trước khi biết nguyên nhân có thể mask vấn đề thật. Nếu lỗi là runtime exception do code bug, tăng resource không fix gì. Tăng resource là giải pháp khi đã xác định bottleneck qua log/metrics — không phải bước đầu tiên.

---

## Câu 5

**Your AI API experiences high latency and log messages indicate CPU throttling during peak traffic. Which change most directly addresses the throttling?**

- Increase the per-replica CPU allocation, and then reassess scaling rules based on throughput. ✅
- Deactivate the newest revision to reduce load on the system.
- Switch the image reference from digest to tag.

**Đáp án: Increase per-replica CPU allocation**

Log message "CPU throttling" là signal rõ ràng: container đang bị giới hạn CPU bởi platform. Giải pháp trực tiếp là tăng CPU limit cho mỗi replica:

```bash
az containerapp update \
    --name ai-api \
    --resource-group rg-aca-demo \
    --cpu 2.0 \
    --memory 4Gi
```

Sau khi tăng CPU, reassess scaling rules — với CPU lớn hơn mỗi replica có thể handle nhiều concurrent request hơn, có thể cần ít replica hơn (giảm cost) hoặc cần điều chỉnh scale-out threshold.

Tại sao các đáp án còn lại sai:
- **Deactivate newest revision:** Nếu newest revision là version đang chạy production và CPU throttling xảy ra dưới peak traffic — deactivate nó sẽ làm hỏng service hoàn toàn. Deactivate không giải quyết CPU throttling, nó chỉ xoá revision khỏi traffic. Hơn nữa, câu hỏi không cho thấy vấn đề liên quan đến revision cụ thể nào.
- **Switch from digest to tag:** Image reference format (digest vs tag) hoàn toàn không liên quan đến CPU throttling hay latency. Đây là về traceability và deployment safety, không ảnh hưởng đến runtime performance.

---

## Tổng kết — Kết quả 5/5

| Câu | Chủ đề | Đáp án |
|---|---|---|
| 1 | Image traceability cho production | Digest (`@sha256:...`) |
| 2 | Ngừng traffic đến revision mà không xoá | `az containerapp revision deactivate` |
| 3 | Nguyên nhân phổ biến nhất khi readiness fail ngay | Sai port hoặc sai path trong probe config |
| 4 | First step khi nghi ngờ runtime exception ở một revision | Stream log + filter by revision |
| 5 | Fix CPU throttling | Tăng per-replica CPU, reassess scaling |

---

*Module hoàn thành. Learning Path ACA tiếp theo: Module 3*
