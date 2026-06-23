# Telemetry Contract Review - CDO-02
**Nguồn:** `templates/ai/contracts/telemetry-contract.md` (do AI team cung cấp)  
**Người review:** CDO-02  
**Mục đích:** Đọc hiểu, tóm tắt và chuẩn bị câu hỏi/đối chất trước meeting với AI team  
**Ngày:** 2026-06-23  

---

## 1. Tóm tắt nhanh (TL;DR)

Contract này xác định **CDO là người sản xuất toàn bộ telemetry data** để AI tiêu thụ. AI không tự thu thập dữ liệu — AI đợi CDO gửi signals đến qua SQS, rồi mới detect/decide/verify.

**Nguồn data thật của CDO-02:** infra EKS sandbox chạy Online Boutique workloads, thu thập qua **Container Insights + Prometheus** (metrics), **CloudWatch Logs** (logs), **OpenTelemetry → X-Ray/Jaeger** (traces). Không dùng CSV offline dataset.

**5 signals CDO phải cung cấp:**

| # | Signal | Loại | Tần suất | SLA gửi | Nguồn thật (CDO-02 infra) |
|---|---|---|---|---|---|
| 1 | `istio_request_error_rate` | Gauge | 5 giây | p99 < 15s | Prometheus/Istio metrics scrape từ EKS — tính `Δerror/Δrequest` |
| 2 | `istio_request_latency_p95` | Gauge | 5 giây | p99 < 15s | Prometheus histogram `istio_request_duration_milliseconds` p95 |
| 3 | `container_memory_working_set_bytes` | Gauge | 10 giây | p99 < 20s | Container Insights metric `container_memory_working_set` |
| 4 | `app_log_error_event` | Event | Real-time | p99 < 5s | CloudWatch Logs — lọc log level ERROR / stack trace từ pod logs |
| 5 | `trace_span_error_event` | Event | Real-time | p99 < 10s | OpenTelemetry traces — lọc span có `status.code = ERROR` |

**2 quy tắc bắt buộc:**
- Mọi payload **phải có** `tenant_id` — map từ Kubernetes namespace (`tenant-a` / `tenant-b`) sang tenant ID tương ứng.
- Timestamp **phải** là RFC3339 UTC, độ chính xác millisecond.

---

## 2. Chi tiết từng signal — nguồn data thật từ CDO-02 infra

### Signal 1 — `istio_request_error_rate`
> **Dùng để làm gì:** AI detect lỗi service (error rate spike) và verify sau khi execute action.

**Nguồn thật:** Prometheus scrape Istio sidecar metrics từ các pod trong EKS sandbox.

**CDO phải làm:**
- Query Prometheus lấy 2 counters: `istio_requests_total{response_code!~"2.."}` (error) và `istio_requests_total` (total) — theo `source_workload` và `destination_service`.
- Tính **rate theo cửa sổ 5 giây**: `rate(error[5s]) / rate(total[5s])`.
- Map `source_workload` → `service`, `destination_service` → `endpoint`.
- Map Kubernetes namespace → `tenant_id` (ví dụ: namespace `tenant-a` → `tenant_id: "tenant-a"`).
- Emit qua SQS với schema chuẩn.

**Ví dụ output:**
```json
{
  "ts": "2026-06-25T10:30:00.123Z",
  "tenant_id": "tenant-a",
  "service": "adservice",
  "endpoint": "hipstershop.AdService/GetAds",
  "value": 0.45,
  "labels": { "system": "OB", "deployment_version": "v2.3.1" }
}
```
> `0.45` = 45% requests bị lỗi trong 5 giây vừa rồi.

**Điểm kỹ thuật:** Đây là **derived metric** — tính từ 2 Prometheus counters, không đọc thẳng. Khi pod restart, counter reset về 0 → delta âm, cần skip data point hoặc gắn flag `counter_reset: true`.

---

### Signal 2 — `istio_request_latency_p95`
> **Dùng để làm gì:** AI detect service stuck hoặc latency spike.

**Nguồn thật:** Prometheus histogram `istio_request_duration_milliseconds` từ Istio sidecar.

**CDO phải làm:**
- Query Prometheus: `histogram_quantile(0.95, rate(istio_request_duration_milliseconds_bucket[5m]))` theo `destination_workload` và `destination_service`.
- Convert đơn vị từ seconds sang milliseconds nếu cần (Istio mặc định là seconds).
- Emit mỗi 5 giây qua SQS.

**Ví dụ output:**
```json
{
  "ts": "2026-06-25T10:30:00.123Z",
  "tenant_id": "tenant-b",
  "service": "checkoutservice",
  "endpoint": "hipstershop.CheckoutService/PlaceOrder",
  "value": 245.0,
  "labels": { "system": "OB", "deployment_version": "v1.0.4" }
}
```
> `245.0` ms = latency p95.

**Lưu ý:** Prometheus histogram_quantile yêu cầu đủ bucket resolution. Nếu Istio dùng default buckets, p95 sẽ là estimate. Chấp nhận được cho capstone.

---

### Signal 3 — `container_memory_working_set_bytes`
> **Dùng để làm gì:** AI detect memory leak hoặc OOM prevention.

**Nguồn thật:** CloudWatch Container Insights metric `container_memory_working_set` hoặc Prometheus `container_memory_working_set_bytes` từ cAdvisor (EKS node).

**CDO phải làm:**
- Pull từ CloudWatch Metrics API: namespace `ContainerInsights`, metric `container_memory_working_set`, dimension `ClusterName / Namespace / PodName / ContainerName`.
- Hoặc query Prometheus: `container_memory_working_set_bytes{namespace="tenant-a", container!="POD"}`.
- Map `PodName` → `pod_name`, `ContainerName` → `container`, `Namespace` → `tenant_id`.
- Emit mỗi 10 giây qua SQS.

**Ví dụ output:**
```json
{
  "ts": "2026-06-25T10:30:00.000Z",
  "tenant_id": "tenant-a",
  "service": "emailservice",
  "pod_name": "emailservice-68d7f5c9b-abcde",
  "container": "main",
  "value": 419430400,
  "labels": { "system": "OB" }
}
```
> `419430400` bytes ≈ 400 MB.

**Điểm kỹ thuật:** Schema có thêm `pod_name` và `container`, không có `endpoint` — khác 2 signal trên. Nếu dùng Container Insights, chú ý resolution mặc định là 60 giây; cần bật high-resolution hoặc dùng Prometheus scrape để đạt 10 giây.

---

### Signal 4 — `app_log_error_event`
> **Dùng để làm gì:** AI phân tích stack trace để chẩn đoán code-level fault trong Online Boutique services.

**Nguồn thật:** CloudWatch Logs — pod logs của các Online Boutique services được thu thập qua Fluent Bit / CloudWatch Container Insights log driver trên EKS.

**CDO phải làm:**
- Đặt CloudWatch Logs subscription filter hoặc Lambda trigger trên log group `/aws/containerinsights/<cluster>/application`.
- Filter pattern: `ERROR` hoặc `Exception` hoặc `Traceback`.
- Parse log record để extract: `service` (từ pod label), `pod_name`, `level`, `message` (stack trace).
- **Bắt buộc:** redact/lọc PII trước khi emit (email, phone, user ID trong log message).
- Emit real-time qua SQS với SLA p99 < 5 giây.

**Ví dụ output:**
```json
{
  "ts": "2026-06-25T10:30:05.456Z",
  "tenant_id": "tenant-a",
  "service": "adservice",
  "pod_name": "adservice-5f8d9b7c-xyz12",
  "level": "ERROR",
  "message": "java.lang.NullPointerException: Cannot invoke 'String.length()' because 'param' is null\n\tat com.hipstershop.adservice.AdService.getAds(AdService.java:45)",
  "labels": { "system": "OB" }
}
```

**Điểm kỹ thuật:** Signal là **Event** — chỉ emit khi có lỗi, không theo interval. SLA p99 < 5 giây là nghiêm nhất trong 5 signals. CloudWatch Logs subscription filter có thể có độ trễ vài giây — cần test SLA thực tế.

---

### Signal 5 — `trace_span_error_event`
> **Dùng để làm gì:** AI chẩn đoán lỗi liên dịch vụ và tìm điểm phát sinh lỗi đầu tiên trong call chain Online Boutique.

**Nguồn thật:** OpenTelemetry Collector chạy trên EKS, thu traces từ các Online Boutique services (instrumented với OTel SDK hoặc Istio telemetry), export sang AWS X-Ray hoặc Jaeger.

**CDO phải làm:**
- OTel Collector nhận spans từ services.
- Filter span có `status.code = ERROR` (OTel status code) hoặc HTTP/gRPC status != 2xx/OK.
- Extract: `service.name` → `service`, `span.operation` → `operation`, `trace_id`, `span_id`, `status_code`, `duration`.
- Map `k8s.namespace.name` → `tenant_id`.
- Emit real-time qua SQS.

**Ví dụ output:**
```json
{
  "ts": "2026-06-25T10:30:04.999Z",
  "tenant_id": "tenant-b",
  "service": "frontend",
  "operation": "grpc.hipstershop.ProductCatalogService/GetProduct",
  "trace_id": "d472bd0a6bda79d8d0b2852d8165cb97",
  "span_id": "cc3118e92762c87f",
  "status_code": 2.0,
  "duration_ms": 150.5,
  "labels": { "system": "OB" }
}
```

**Điểm kỹ thuật:** Phụ thuộc Online Boutique services có được instrument OTel không. Một số services (Go, Python, Java) có OTel auto-instrumentation; một số cần manual. Đây là signal có dependency cao nhất — cần xác nhận mức instrument thực tế trước W12.

---

## 3. Mapping signal → Pattern CDO đang build

Đây là phần liên kết contract AI với 3 patterns CDO-02 đã chốt, **dùng data thật từ EKS sandbox**:

| Pattern CDO | Signals cần | Nguồn thật trên EKS |
|---|---|---|
| Service stuck / latency spike | `istio_request_latency_p95` + `istio_request_error_rate` | Prometheus scrape Istio sidecars |
| Error rate spike / code-level fault | `istio_request_error_rate` + `app_log_error_event` + `trace_span_error_event` | Prometheus metrics + CloudWatch Logs + OTel traces |
| Memory pressure / OOM prevention | `container_memory_working_set_bytes` | Container Insights hoặc cAdvisor via Prometheus |

> **Nhận xét:** 3 patterns CDO đã chọn đều được cover đủ bởi 5 signals. Điều kiện để data thật hoạt động là **Istio phải được enable** trên EKS cluster và **OTel Collector** phải được deploy.

---

## 4. Telemetry Pipeline Architecture (data thật)

```
EKS Sandbox — Online Boutique workloads
        │
        ├─ Istio sidecars
        │     └─ Prometheus scrape (5s)
        │           ├─ → Signal 1: istio_request_error_rate   ─┐
        │           └─ → Signal 2: istio_request_latency_p95  ─┤
        │                                                       │
        ├─ cAdvisor / Container Insights                        ├─► Telemetry
        │     └─ Prometheus / CloudWatch Metrics (10s)          │   Collector
        │           └─ → Signal 3: container_memory_working_set─┤   (CDO)
        │                                                       │     │
        ├─ Fluent Bit → CloudWatch Logs                         │     │
        │     └─ Subscription Filter / Lambda                   │     ▼
        │           └─ → Signal 4: app_log_error_event        ─┤   SQS Queue
        │                                                       │     │
        └─ OTel Collector → X-Ray / Jaeger                      │     ▼
              └─ Span processor (filter ERROR)                  │   AI Engine
                    └─ → Signal 5: trace_span_error_event     ─┘   /v1/detect
```

**Thành phần CDO cần deploy trên EKS:**

| Component | Technology | Mục đích |
|---|---|---|
| Prometheus | `kube-prometheus-stack` Helm chart | Scrape Istio + cAdvisor metrics |
| Istio | Istio service mesh | Cung cấp `istio_requests_total`, `istio_request_duration` |
| Fluent Bit | DaemonSet trên EKS node | Collect pod logs → CloudWatch Logs |
| OTel Collector | Deployment trong `platform` namespace | Collect traces → X-Ray/Jaeger, filter error spans |
| Telemetry Collector | CDO custom service | Normalize + inject `tenant_id` + gửi SQS |

---

## 5. Điểm nổi bật / highlight cần nhớ

### ✅ CDO-02 đang làm đúng
- Requirements doc đã liệt kê đúng 5 signals khớp contract.
- Infra design đã chọn đúng stack: Prometheus + CloudWatch + OTel khớp hoàn toàn với nguồn data cần thiết.
- Đã ghi PII anonymization là trách nhiệm của CDO.
- Tenant isolation theo namespace — map thẳng từ `k8s namespace` → `tenant_id`.

### ⚠️ Điểm kỹ thuật CDO cần chú ý khi implement

1. **Istio phải được enable trên EKS** — Signal 1 và 2 hoàn toàn phụ thuộc Istio sidecar. Nếu Istio chưa được cài hoặc chưa inject sidecar vào Online Boutique pods, 2 signals này không có data. **Đây là dependency quan trọng nhất cần confirm trước W12.**

2. **Signal 1 cần tính rate, không đọc thẳng** — Prometheus counter `istio_requests_total` là cumulative, phải dùng `rate()` hoặc tính delta thủ công theo cửa sổ 5 giây. Khi pod restart, counter reset — cần Prometheus `increase()` hoặc skip reset point.

3. **Container Insights resolution** — Default CloudWatch Container Insights scrape mỗi 60 giây, không đủ cho SLA 10 giây của Signal 3. Cần bật **enhanced monitoring** hoặc chuyển sang Prometheus scrape cAdvisor trực tiếp (mặc định 15s, có thể cấu hình thấp hơn).

4. **OTel instrumentation của Online Boutique** — Signal 5 cần Online Boutique services emit traces. Một số services có OTel SDK tích hợp sẵn; một số cần thêm auto-instrumentation agent. Cần kiểm tra từng service trong sandbox.

5. **Tenant ID mapping** — Không còn inject `tnt-re2-simulation` / `tnt-re3-simulation`. Thay vào đó, map từ Kubernetes namespace label: `namespace: tenant-a` → `tenant_id: "tenant-a"`. Telemetry Collector phải đọc namespace label từ pod metadata.

6. **Schema không đồng nhất giữa các signals** — Signal 3 có thêm `pod_name`, `container`, không có `endpoint`. Signal 5 có `trace_id`, `span_id`, `status_code`, `duration_ms`. Pipeline CDO cần xử lý riêng từng signal type.

7. **SLA log event p99 < 5 giây** — CloudWatch Logs subscription filter có thể có độ trễ 2–10 giây. Nếu SLA nghiêm, xem xét dùng Fluent Bit → Kinesis Data Streams → Lambda thay vì subscription filter để giảm latency.

---

## 6. Câu hỏi cần đối chất với AI team trong meeting

Dựa trên review contract và infra thật của CDO-02, các điểm dưới đây cần làm rõ:

### 6.1 Về tenant_id — mapping từ Kubernetes namespace

> **Q1:** CDO-02 dùng Kubernetes namespace-based isolation (`tenant-a`, `tenant-b`). `tenant_id` trong payload sẽ map 1:1 từ namespace. AI có accept `tenant_id: "tenant-a"` không, hay cần format khác (`tnt-a`, UUID)? Cần bảng mapping chính thức để CDO không gửi sai.

### 6.2 Về Istio requirement

> **Q2:** Signals 1 và 2 hoàn toàn phụ thuộc Istio sidecar (`istio_requests_total`, `istio_request_duration_milliseconds`). Nếu EKS sandbox chưa enable Istio, CDO không có data thật cho 2 signals này. AI team có thể dùng Prometheus metrics Kubernetes native (không qua Istio) không? Ví dụ: `kube_pod_container_status_restarts_total` + HTTP metrics từ app? Hay bắt buộc phải có Istio?

### 6.3 Về SQS endpoint và format

> **Q3:** Contract nói CDO gửi signals qua SQS. Cần AI team cung cấp: SQS queue ARN, cấu trúc message (raw JSON hay wrapped envelope), có 1 queue chung hay queue riêng theo signal type? CDO không thể implement pipeline mà không có thông tin này.

### 6.4 Về OTel instrumentation — Online Boutique

> **Q4:** Signal 5 (`trace_span_error_event`) cần Online Boutique services emit OTel traces. AI team có biết services nào trong Online Boutique đã có OTel SDK không? CDO cần biết để plan instrumentation cho W12. Nếu AI team đã có OTel setup, CDO có thể reuse collector config không?

### 6.5 Về boundary AI/CDO trên Kubernetes (carry-over)

> **Q5:** Deployment contract nhắc AI Task Role có thể fetch kubeconfig/gọi EKS API. CDO-02 đã design rõ: AI chỉ trả `action_plan`, CDO executor mới mutate Kubernetes. Cần AI team confirm: AI có execute action trực tiếp không, hay chỉ trả `action_plan`? Nếu AI giữ kubeconfig, RBAC boundary giữa 2 bên là gì?

### 6.6 Về PII anonymization spec

> **Q6:** CDO phải lọc PII trong log trước khi gửi Signal 4. AI team có yêu cầu cụ thể về cách anonymize không? Redact hoàn toàn field hay chỉ mask? AI có cần stack trace đầy đủ để diagnose không, hay có thể truncate message dài?

### 6.7 Về Container Insights resolution cho Signal 3

> **Q7:** SLA của `container_memory_working_set_bytes` là p99 < 20 giây với tần suất 10 giây. Container Insights mặc định scrape 60 giây. CDO dự kiến dùng Prometheus + cAdvisor để đạt tần suất này. AI team có yêu cầu gì về nguồn metric cụ thể không, hay chỉ cần đúng schema và tần suất?

---

## 7. Checklist trước meeting

- [x] Đọc và hiểu 5 signals trong contract
- [x] Map signals → sources thật trên EKS infra CDO-02
- [x] Xác định gaps / điểm kỹ thuật cần làm rõ
- [x] Chuẩn bị câu hỏi đối chất với AI team
- [ ] Confirm với AI team: tenant_id format (Q1)
- [ ] Confirm với AI team: Istio requirement hay có alternative? (Q2)
- [ ] Confirm với AI team: SQS endpoint/config (Q3)
- [ ] Confirm với AI team: OTel instrumentation status của Online Boutique (Q4)
- [ ] Confirm với AI team: AI boundary / kubeconfig (Q5)
- [ ] Confirm với AI team: PII anonymization spec (Q6)

---

## 8. Điểm tóm tắt 1 câu để present với team

> **"Contract này nói CDO là data producer: thu 5 signals từ EKS sandbox (Istio metrics qua Prometheus, pod logs qua CloudWatch, traces qua OTel), normalize, inject tenant_id từ Kubernetes namespace, rồi push qua SQS cho AI tiêu thụ. 3 patterns CDO đang build đều được cover đủ. Điều kiện tiên quyết là Istio phải enable trên EKS. Trước meeting cần chốt: SQS config, tenant_id format, Istio requirement, và AI boundary trên Kubernetes."**
