# Telemetry Contract Review - CDO-02
**Nguồn:** `templates/ai/contracts/telemetry-contract.md` (do AI team cung cấp)  
**Người review:** CDO-02  
**Mục đích:** Đọc hiểu, tóm tắt và chuẩn bị câu hỏi/đối chất trước meeting với AI team  
**Ngày:** 2026-06-23  

---

## 1. Tóm tắt nhanh (TL;DR)

Contract này xác định **CDO là người sản xuất toàn bộ telemetry data** để AI tiêu thụ. AI không tự thu thập dữ liệu — AI đợi CDO gửi signals đến qua SQS, rồi mới detect/decide/verify.

**5 signals CDO phải cung cấp:**

| # | Signal | Loại | Tần suất | SLA gửi | Nguồn gốc data |
|---|---|---|---|---|---|
| 1 | `istio_request_error_rate` | Gauge | 5 giây | p99 < 15s | Tính từ `metrics.csv` (RE3) |
| 2 | `istio_request_latency_p95` | Gauge | 5 giây | p99 < 15s | Đọc trực tiếp `metrics.csv` (RE3) |
| 3 | `container_memory_working_set_bytes` | Gauge | 10 giây | p99 < 20s | Đọc trực tiếp `metrics.csv` (RE3) |
| 4 | `app_log_error_event` | Event | Real-time | p99 < 5s | Parse `logs.csv` (lọc ERROR level) |
| 5 | `trace_span_error_event` | Event | Real-time | p99 < 10s | Parse `traces.csv` (lọc `statusCode != 0.0`) |

**2 quy tắc bắt buộc:**
- Mọi payload **phải có** `tenant_id` (inject `tnt-re2-simulation` hoặc `tnt-re3-simulation` khi dùng offline dataset).
- Timestamp **phải** là RFC3339 UTC, độ chính xác millisecond.

---

## 2. Chi tiết từng signal

### Signal 1 — `istio_request_error_rate`
> **Dùng để làm gì:** AI detect lỗi service (error rate spike) và verify sau khi execute action.

**CDO phải làm:**
- Đọc 2 counter từ `metrics.csv`: `<service>_istio-error-total` và `<service>_istio-request-total`.
- Tính **delta** (chênh lệch) của cả 2 counter theo từng cửa sổ 5 giây.
- Chia delta error / delta request để ra tỷ lệ lỗi (0.0 → 1.0).
- Emit qua SQS với schema chuẩn.

**Ví dụ output:**
```json
{
  "ts": "2026-06-25T10:30:00.123Z",
  "tenant_id": "tnt-re3-simulation",
  "service": "adservice",
  "endpoint": "hipstershop.AdService/GetAds",
  "value": 0.45
}
```
> `0.45` = 45% requests bị lỗi.

**Điểm kỹ thuật cần nhớ:** Đây là phép tính **derived metric** — CDO tính từ 2 counters, không phải đọc trực tiếp. Nếu counter bị reset (pod restart), cần xử lý edge case này.

---

### Signal 2 — `istio_request_latency_p95`
> **Dùng để làm gì:** AI detect service stuck hoặc latency spike.

**CDO phải làm:**
- Đọc trực tiếp cột `<service>_istio-latency-95` từ `metrics.csv`.
- Emit mỗi 5 giây qua SQS.

**Ví dụ output:**
```json
{
  "ts": "2026-06-25T10:30:00.123Z",
  "tenant_id": "tnt-re2-simulation",
  "service": "checkoutservice",
  "endpoint": "hipstershop.CheckoutService/PlaceOrder",
  "value": 245.0
}
```
> `245.0` ms = latency p95 tại thời điểm đó.

**Lưu ý:** Signal này đơn giản nhất — chỉ đọc + forward. Không cần tính toán.

---

### Signal 3 — `container_memory_working_set_bytes`
> **Dùng để làm gì:** AI detect memory leak hoặc OOM prevention.

**CDO phải làm:**
- Đọc trực tiếp cột `<service>_container-memory-working-set-bytes` từ `metrics.csv`.
- Emit mỗi 10 giây (tần suất thấp hơn signal 1 và 2).
- Schema có thêm trường `pod_name` và `container`.

**Ví dụ output:**
```json
{
  "ts": "2026-06-25T10:30:00.000Z",
  "tenant_id": "tnt-re2-simulation",
  "service": "emailservice",
  "pod_name": "emailservice-68d7f5c9b-abcde",
  "container": "main",
  "value": 419430400
}
```
> `419430400` bytes ≈ 400 MB.

**Điểm kỹ thuật:** Schema khác 2 signal trên — có thêm `pod_name` và `container`, không có `endpoint`. Cần chú ý mapping đúng field.

---

### Signal 4 — `app_log_error_event`
> **Dùng để làm gì:** AI phân tích stack trace để chẩn đoán code-level fault (f1–f5 trong dataset RE3).

**CDO phải làm:**
- Đọc `logs.csv`.
- Lọc các dòng có `level = ERROR` hoặc có stack trace.
- Parse và emit real-time qua SQS.
- **Bắt buộc:** lọc/mã hóa PII trước khi gửi.

**Ví dụ output:**
```json
{
  "ts": "2026-06-25T10:30:05.456Z",
  "tenant_id": "tnt-re3-simulation",
  "service": "adservice",
  "pod_name": "adservice-5f8d9b7c-xyz12",
  "level": "ERROR",
  "message": "java.lang.NullPointerException: Cannot invoke 'String.length()' because 'param' is null\n\tat com.hipstershop.adservice.AdService.getAds(AdService.java:45)"
}
```

**Điểm kỹ thuật:** Signal này là **Event** (không phải Gauge), nghĩa là chỉ emit khi có lỗi, không phải theo interval cố định. SLA nghiêm nhất trong 5 signals: p99 < 5 giây.

---

### Signal 5 — `trace_span_error_event`
> **Dùng để làm gì:** AI chẩn đoán lỗi liên dịch vụ và tìm điểm phát sinh lỗi đầu tiên.

**CDO phải làm:**
- Đọc `traces.csv`.
- Lọc các span có `statusCode != 0.0`.
- Parse và emit real-time qua SQS.

**Ví dụ output:**
```json
{
  "ts": "2026-06-25T10:30:04.999Z",
  "tenant_id": "tnt-re2-simulation",
  "service": "frontend",
  "operation": "grpc.hipstershop.ProductCatalogService/GetProduct",
  "trace_id": "d472bd0a6bda79d8d0b2852d8165cb97",
  "span_id": "cc3118e92762c87f",
  "status_code": 2.0,
  "duration_ms": 150.5
}
```

**Điểm kỹ thuật:** Schema phức tạp nhất, có `trace_id`, `span_id`, `status_code`, `duration_ms`. Filter điều kiện là `statusCode != 0.0` (float, không phải int).

---

## 3. Mapping signal → Pattern CDO đang build

Đây là phần liên kết contract AI với 3 patterns CDO-02 đã chốt:

| Pattern CDO (từ req doc) | Signals cần | Nguồn data |
|---|---|---|
| Service stuck / latency spike | `istio_request_latency_p95` + `istio_request_error_rate` | RE3 `metrics.csv` |
| Error rate spike / code-level fault | `istio_request_error_rate` + `app_log_error_event` + `trace_span_error_event` | RE3 `metrics.csv`, `logs.csv`, `traces.csv` |
| Memory pressure / OOM prevention | `container_memory_working_set_bytes` | RE3 `metrics.csv` |

> **Nhận xét:** 3 patterns CDO đã chọn đều được cover đủ bởi 5 signals trong contract. Không có gap signal nào.

---

## 4. Điểm nổi bật / highlight cần nhớ

### ✅ CDO-02 đang làm đúng
- Requirements doc đã liệt kê đúng 5 signals khớp contract.
- Đã ghi rõ `tenant_id` inject rule (`tnt-re2-simulation` / `tnt-re3-simulation`).
- Đã map signal → pattern build đúng.
- Đã ghi PII anonymization là trách nhiệm của CDO.

### ⚠️ Điểm kỹ thuật CDO cần chú ý khi implement

1. **Signal 1 cần tính delta** — không đọc thẳng counter, phải tính chênh lệch giữa 2 lần đọc. Nếu counter reset (pod crash/restart), giá trị delta có thể âm → cần xử lý edge case này.

2. **Tenant ID inject** — dataset RE2/RE3 gốc không có `tenant_id`. CDO phải inject vào mọi payload trước khi gửi. Contract nói rõ RE2 → `tnt-re2-simulation`, RE3 → `tnt-re3-simulation`. Cần hỏi AI team: **khi nào dùng RE2, khi nào dùng RE3?** (vì hiện tại trong schema examples bị trộn lẫn nhau).

3. **SLA emit khác nhau theo signal** — signal log/trace event có SLA khắt khe hơn (5s, 10s) so với metric gauge (15s, 20s). Pipeline CDO cần ưu tiên log/trace event.

4. **Schema không đồng nhất giữa các signals** — `container_memory_working_set_bytes` có thêm `pod_name`, `container` mà không có `endpoint`. `trace_span_error_event` có thêm `trace_id`, `span_id`, `status_code`, `duration_ms`. Cần implement riêng cho từng signal type, không dùng chung 1 schema.

5. **Trường `labels` là optional nhưng nên có** — trong examples đều có `labels: { system: "OB" }`. Với W12, nên gắn thêm `deployment_version` để AI có thêm context.

---

## 5. Câu hỏi cần đối chất với AI team trong meeting

Dựa trên review contract, các điểm dưới đây CDO-02 cần làm rõ:

### 5.1 Về tenant_id và nguồn dataset

> **Q1:** Trong contract, `istio_request_latency_p95` schema example dùng `tnt-re2-simulation`, nhưng chú thích nói nguồn data là RE3. `istio_request_error_rate` schema example dùng `tnt-re3-simulation` và nguồn là RE3. Vậy rule inject `tenant_id` theo nguồn dataset (RE2 vs RE3) hay theo signal type? Cần table mapping rõ ràng để CDO không inject sai.

### 5.2 Về Signal 1 — counter delta edge case

> **Q2:** Với `istio_request_error_rate`, khi pod restart khiến counter reset về 0, delta sẽ cho giá trị âm hoặc spike bất thường. CDO nên xử lý thế nào: bỏ qua data point đó, emit 0, hay gắn flag đặc biệt? Contract chưa đề cập.

### 5.3 Về "cửa sổ trượt" (sliding window) cho Signal 1

> **Q3:** Contract nói Signal 1 dùng "cửa sổ trượt 5 giây". CDO hiểu là lấy delta trong 5 giây gần nhất, tính liên tục. Có đúng không? Hay "cửa sổ trượt" ở đây có nghĩa khác (tumbling window)?

### 5.4 Về SQS endpoint

> **Q4:** Contract nói CDO gửi signals qua SQS. Nhưng không có thông tin về: SQS queue ARN, queue per signal hay chung 1 queue, message format (raw JSON hay wrapped), và authentication. Cần AI team cung cấp để CDO implement đúng.

### 5.5 Về AI Endpoint → xác nhận lại boundary

> **Q5 (nhắc lại từ requirements doc):** Contract Deployment nói AI Task Role có thể fetch kubeconfig. Nếu AI đã có kubeconfig, AI có execute action trực tiếp không? Hay AI chỉ trả `action_plan` và CDO executor là người duy nhất mutate Kubernetes? Boundary này cần chốt rõ vì ảnh hưởng trực tiếp đến thiết kế RBAC của CDO.

### 5.6 Về PII anonymization

> **Q6:** Contract yêu cầu CDO lọc PII trong logs trước khi gửi. AI team có yêu cầu cụ thể cách anonymize không (hash, redact, hoặc chỉ cần không gửi field nhạy cảm)? Và AI có cần giữ nguyên stack trace đầy đủ để diagnose không, hay có thể redact partial?

### 5.7 Về Offline Simulation — mock execute hay action thật?

> **Q7:** Với Offline Simulation Mode (dùng RE2/RE3 dataset), CDO chỉ cần inject tenant_id và gửi data từ file CSV? Hay CDO còn cần simulate Kubernetes behavior (mock action responses) để AI có thể verify? Evidence W12 sẽ là mock action hay action thật trên EKS sandbox?

---

## 6. Checklist trước meeting

- [x] Đọc và hiểu 5 signals trong contract
- [x] Map signals → patterns CDO đang build
- [x] Xác định gaps / điểm kỹ thuật cần làm rõ
- [x] Chuẩn bị câu hỏi đối chất với AI team
- [ ] Confirm với AI team: tenant_id injection rule (Q1)
- [ ] Confirm với AI team: SQS endpoint details (Q4)
- [ ] Confirm với AI team: AI boundary / kubeconfig (Q5)
- [ ] Confirm với AI team: PII anonymization spec (Q6)
- [ ] Confirm với AI team: Offline simulation mode (Q7)

---

## 7. Điểm tóm tắt 1 câu để present với team

> **"Contract này nói CDO là data producer: đọc RE2/RE3 dataset, tính toán/parse, inject tenant_id, rồi push 5 loại signals lên SQS để AI tiêu thụ. 3 patterns CDO đang build đều được cover đủ. Trước meeting cần chốt 7 điểm kỹ thuật với AI team, đặc biệt là SQS endpoint config, tenant_id injection rule, và AI boundary trên Kubernetes."**
