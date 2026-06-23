# Telemetry Data Pipeline — Lấy Data Thật từ RE2/RE3 Dataset
**Doc owner:** CDO-02  
**Trạng thái:** Draft W11  
**Cập nhật lần cuối:** 2026-06-23  

> **Mục tiêu của file này:** Hướng dẫn cụ thể cách CDO-02 dùng **RE2/RE3 dataset** làm nguồn dữ liệu thật cho EKS sandbox, xây pipeline đọc → xử lý → emit 5 signals theo Telemetry Contract, thay vì mock hay sinh data giả.

---

## 1. Tại sao dùng RE2/RE3 thay vì mock?

| Mock data | RE2/RE3 dataset |
|---|---|
| Dễ viết nhưng AI không "thấy" lỗi thật | Chứa các fault patterns thật của Online Boutique |
| Không có stack trace, trace ID thật | Có đủ `metrics.csv`, `logs.csv`, `traces.csv` |
| Demo kém thuyết phục | AI có thể detect anomaly thật và trả action plan có ý nghĩa |
| Không chứng minh pipeline thật | Evidence W12 mạnh hơn vì data có nguồn gốc rõ |

RE2 = **Resource & Network Faults** (memory pressure, latency spike, network fault)  
RE3 = **Code-Level Faults** (NullPointerException, error rate spike, trace error)

---

## 2. Cấu trúc data RE2/RE3

Dataset gồm 3 loại file, tương ứng 3 loại signal trong contract:

```
re2-dataset/
  metrics.csv     ← Signal 1, 2, 3 (gauge metrics theo time series)
  logs.csv        ← Signal 4 (application logs, có ERROR level)
  traces.csv      ← Signal 5 (distributed traces, có statusCode)

re3-dataset/
  metrics.csv
  logs.csv
  traces.csv
```

### 2.1 Cột quan trọng trong `metrics.csv`

| Cột trong file | Map sang Signal | Ghi chú |
|---|---|---|
| `<service>_istio-error-total` | Signal 1 (counter — cần tính delta) | Counter, tăng dần |
| `<service>_istio-request-total` | Signal 1 (counter — cần tính delta) | Counter, tăng dần |
| `<service>_istio-latency-95` | Signal 2 (đọc thẳng) | Gauge, ms |
| `<service>_container-memory-working-set-bytes` | Signal 3 (đọc thẳng) | Gauge, bytes |

### 2.2 Cột quan trọng trong `logs.csv`

| Cột trong file | Điều kiện lọc | Map sang Signal |
|---|---|---|
| `level` | `= "ERROR"` hoặc có stack trace | Signal 4 |
| `message` | Chứa exception/stack trace | Signal 4 |
| `service`, `pod_name`, `timestamp` | Bắt buộc trong schema | Signal 4 |

### 2.3 Cột quan trọng trong `traces.csv`

| Cột trong file | Điều kiện lọc | Map sang Signal |
|---|---|---|
| `statusCode` | `!= 0.0` | Signal 5 |
| `trace_id`, `span_id` | Bắt buộc trong schema | Signal 5 |
| `service`, `operation`, `duration_ms` | Context cho AI diagnose | Signal 5 |

---

## 3. Kiến trúc Pipeline — Toàn cảnh

```
RE2/RE3 Dataset (S3 hoặc local volume trong EKS)
        │
        ▼
┌─────────────────────────────────────────────────┐
│           CDO Telemetry Preprocessor            │
│           (Pod trong namespace: platform)        │
│                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌────────┐ │
│  │ Metrics      │  │ Log Parser   │  │ Trace  │ │
│  │ Reader       │  │              │  │ Parser │ │
│  │ (Signal 1,2,3│  │ (Signal 4)   │  │(Signal5│ │
│  └──────┬───────┘  └──────┬───────┘  └───┬────┘ │
│         │                 │              │       │
│         └─────────────────┴──────────────┘       │
│                           │                      │
│              ┌────────────▼──────────────┐        │
│              │   Tenant ID Injector      │        │
│              │   + PII Filter            │        │
│              │   + Schema Validator      │        │
│              └────────────┬──────────────┘        │
└───────────────────────────┼─────────────────────┘
                            │
                            ▼
                     AWS SQS Queue
                            │
                            ▼
                    AI Engine (ECS Fargate)
               https://ai-engine.tf-3.internal/
```

**Tất cả chạy trong EKS sandbox, namespace `platform`**, cùng VPC với AI Engine, gọi qua internal ALB.

---

## 4. Chi tiết từng bước xử lý

### Bước 1 — Đưa dataset vào EKS sandbox

**Cách 1 (khuyến nghị cho W12 demo):** Mount dataset từ S3 vào pod qua `initContainer` hoặc dùng S3 CSI driver.

```yaml
# Ví dụ: ConfigMap chứa đường dẫn dataset
apiVersion: v1
kind: ConfigMap
metadata:
  name: dataset-config
  namespace: platform
data:
  RE2_PATH: "s3://cdo-02-capstone/re2-dataset/"
  RE3_PATH: "s3://cdo-02-capstone/re3-dataset/"
  DATASET_MODE: "offline_simulation"
```

**Cách 2 (nhanh hơn cho W11 test):** Copy file CSV vào PersistentVolume và mount vào pod.

```yaml
volumes:
  - name: dataset-volume
    persistentVolumeClaim:
      claimName: re2-re3-dataset-pvc
```

---

### Bước 2 — Metrics Reader (Signal 1, 2, 3)

**Nhiệm vụ:** Đọc `metrics.csv` theo time order, emit signal mỗi 5–10 giây theo đúng tần suất contract.

**Logic xử lý Signal 1 — `istio_request_error_rate` (phức tạp nhất):**

```
Đọc row N:   error_total = 150,  request_total = 1000
Đọc row N+1: error_total = 165,  request_total = 1050

delta_error   = 165 - 150 = 15
delta_request = 1050 - 1000 = 50
error_rate    = 15 / 50 = 0.30

→ emit value = 0.30
```

**Edge case phải xử lý:**
- `delta_request == 0` → bỏ qua row đó, không emit (tránh chia cho 0).
- Counter reset (giá trị giảm đột ngột) → coi như delta = 0, ghi warning log.
- Thiếu cột trong file → log error, skip row.

**Logic Signal 2 và 3:** Đọc thẳng cột tương ứng, không cần tính toán.

**Output mỗi lần emit (Signal 1 ví dụ):**

```json
{
  "ts": "2026-06-25T10:30:00.000Z",
  "tenant_id": "tnt-re3-simulation",
  "service": "adservice",
  "endpoint": "hipstershop.AdService/GetAds",
  "value": 0.30,
  "labels": {
    "system": "OB",
    "deployment_version": "v2.3.1",
    "dataset_source": "RE3"
  }
}
```

---

### Bước 3 — Log Parser (Signal 4)

**Nhiệm vụ:** Đọc `logs.csv`, lọc các dòng ERROR, parse và emit real-time.

**Filter condition:**
```
level == "ERROR"  OR  message contains ("Exception", "Traceback", "stack trace", "at com.", "at org.")
```

**PII filter — bắt buộc theo security design:**
```
Trước khi emit, scan message:
- Email pattern (.*@.*\..*)        → redact → "[REDACTED_EMAIL]"
- Phone pattern (\d{10,11})        → redact → "[REDACTED_PHONE]"
- Credit card pattern (\d{4}-...) → redact → "[REDACTED_CC]"
```

**Output (Signal 4):**

```json
{
  "ts": "2026-06-25T10:30:05.456Z",
  "tenant_id": "tnt-re3-simulation",
  "service": "adservice",
  "pod_name": "adservice-5f8d9b7c-xyz12",
  "level": "ERROR",
  "message": "java.lang.NullPointerException: Cannot invoke 'String.length()' because 'param' is null\n\tat com.hipstershop.adservice.AdService.getAds(AdService.java:45)",
  "labels": {
    "system": "OB",
    "dataset_source": "RE3"
  }
}
```

---

### Bước 4 — Trace Parser (Signal 5)

**Nhiệm vụ:** Đọc `traces.csv`, lọc span có lỗi, emit real-time.

**Filter condition:**
```
statusCode != 0.0
```

**Lưu ý:** `statusCode` trong file là float — cần so sánh float, không phải int. `statusCode = 0.0` là OK, bất kỳ giá trị khác đều là lỗi.

**Output (Signal 5):**

```json
{
  "ts": "2026-06-25T10:30:04.999Z",
  "tenant_id": "tnt-re2-simulation",
  "service": "frontend",
  "operation": "grpc.hipstershop.ProductCatalogService/GetProduct",
  "trace_id": "d472bd0a6bda79d8d0b2852d8165cb97",
  "span_id": "cc3118e92762c87f",
  "status_code": 2.0,
  "duration_ms": 150.5,
  "labels": {
    "system": "OB",
    "dataset_source": "RE2"
  }
}
```

---

### Bước 5 — Tenant ID Injector

**Quy tắc inject `tenant_id`** (theo Telemetry Contract):

| Nguồn dataset | tenant_id inject |
|---|---|
| RE2 dataset | `tnt-re2-simulation` |
| RE3 dataset | `tnt-re3-simulation` |

Injector đọc biến môi trường `DATASET_SOURCE` (RE2 hoặc RE3) và tự động gắn đúng `tenant_id` vào mọi payload trước khi gửi SQS.

```python
# Pseudocode
TENANT_MAP = {
    "RE2": "tnt-re2-simulation",
    "RE3": "tnt-re3-simulation"
}
payload["tenant_id"] = TENANT_MAP[os.environ["DATASET_SOURCE"]]
```

---

### Bước 6 — Schema Validator + SQS Emit

Trước khi push lên SQS, mỗi payload phải pass schema check:

**Required fields tất cả signals:**
- `ts` — RFC3339 UTC, có millisecond
- `tenant_id` — không được rỗng
- `service` — tên service từ Online Boutique
- `value` hoặc `message` hoặc `status_code` — tuỳ signal type

**Nếu validation fail:**
- Log error với payload gốc (sau khi redact).
- Gửi vào **Dead Letter Queue (DLQ)** riêng — không drop silent.
- Không block pipeline, tiếp tục xử lý row tiếp theo.

**SQS message structure:**
```json
{
  "signal_name": "istio_request_error_rate",
  "payload": { ... },
  "emit_ts": "2026-06-25T10:30:00.000Z",
  "pipeline_version": "v1.0",
  "dataset_source": "RE3"
}
```

---

## 5. Deployment Manifest Sketch

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: telemetry-preprocessor
  namespace: platform
spec:
  replicas: 1
  template:
    spec:
      serviceAccountName: cdo-executor-sa   # IRSA — có quyền gửi SQS, đọc S3
      containers:
        - name: preprocessor
          image: <ecr-repo>/telemetry-preprocessor:v1.0
          env:
            - name: DATASET_SOURCE
              value: "RE3"                   # hoặc RE2, tuỳ scenario
            - name: SQS_QUEUE_URL
              valueFrom:
                secretKeyRef:
                  name: pipeline-config
                  key: sqs_queue_url
            - name: METRICS_EMIT_INTERVAL_SEC
              value: "5"
            - name: MEMORY_EMIT_INTERVAL_SEC
              value: "10"
            - name: DLQ_URL
              valueFrom:
                secretKeyRef:
                  name: pipeline-config
                  key: dlq_url
          volumeMounts:
            - name: dataset-volume
              mountPath: /data/dataset
              readOnly: true
      volumes:
        - name: dataset-volume
          persistentVolumeClaim:
            claimName: re2-re3-dataset-pvc
```

**ServiceAccount `cdo-executor-sa`** cần IAM permissions (qua IRSA):
```
sqs:SendMessage         → SQS queue (signal queue + DLQ)
s3:GetObject            → S3 bucket chứa dataset (nếu dùng S3 mount)
logs:CreateLogStream
logs:PutLogEvents       → CloudWatch Logs
```

---

## 6. Mapping Signal → Dataset → Tenant ID

Bảng này là nguồn chính xác để tham chiếu khi implement và khi AI team hỏi:

| Signal | Dataset nguồn | File | Tenant ID inject | Tần suất | SLA emit |
|---|---|---|---|---|---|
| `istio_request_error_rate` | RE3 | `metrics.csv` | `tnt-re3-simulation` | 5 giây | p99 < 15s |
| `istio_request_latency_p95` | RE3 | `metrics.csv` | `tnt-re3-simulation` | 5 giây | p99 < 15s |
| `container_memory_working_set_bytes` | RE2 | `metrics.csv` | `tnt-re2-simulation` | 10 giây | p99 < 20s |
| `app_log_error_event` | RE3 | `logs.csv` | `tnt-re3-simulation` | Real-time | p99 < 5s |
| `trace_span_error_event` | RE2 | `traces.csv` | `tnt-re2-simulation` | Real-time | p99 < 10s |

> **Lưu ý:** Bảng trên dựa trên phân tích fault type — RE3 chứa code-level faults (error rate, logs, traces lỗi code), RE2 chứa resource/network faults (memory, trace latency). Cần confirm lại với AI team trước W12.

---

## 7. Tenant Isolation trong Pipeline

Theo security design (ADR-003), namespace-based isolation áp dụng cho cả pipeline:

```
Scenario A — test tenant-a:
  DATASET_SOURCE = RE3
  tenant_id inject = "tnt-re3-simulation"
  Tất cả signal gửi qua SQS đều có tenant_id = "tnt-re3-simulation"
  AI detect/decide chỉ trả action cho tenant này
  CDO executor chỉ thao tác namespace "tenant-a"

Scenario B — test tenant-b:
  DATASET_SOURCE = RE2
  tenant_id inject = "tnt-re2-simulation"
  Tất cả signal gửi qua SQS đều có tenant_id = "tnt-re2-simulation"
  CDO executor chỉ thao tác namespace "tenant-b"
```

Cross-tenant test: chạy 2 scenario song song, verify safety gate chặn đúng khi `action_plan` target sai namespace.

---

## 8. Observability cho chính Pipeline

Pipeline cũng cần có observability để CDO-02 debug và chứng minh pipeline chạy đúng:

| Metric | Mô tả | Nơi emit |
|---|---|---|
| `signals_emitted_total` | Counter tổng signal đã gửi SQS | CloudWatch / Prometheus |
| `signals_failed_total` | Counter signal bị reject / vào DLQ | CloudWatch / Prometheus |
| `emit_latency_ms` | Thời gian từ khi đọc row đến khi SQS confirm | CloudWatch |
| `dataset_rows_processed` | Bao nhiêu row đã xử lý | CloudWatch |
| `pii_redacted_total` | Bao nhiêu lần PII bị redact | CloudWatch (audit) |

Log format cho mỗi emit (ghi vào CloudWatch Logs):
```json
{
  "ts": "...",
  "level": "INFO",
  "event": "signal_emitted",
  "signal_name": "istio_request_error_rate",
  "tenant_id": "tnt-re3-simulation",
  "service": "adservice",
  "sqs_message_id": "abc123",
  "correlation_id": "..."
}
```

---

## 9. Checklist Implementation W12

### Phase 1 — Setup (T1-T2 W12)
- [ ] Upload RE2/RE3 dataset lên S3 bucket `cdo-02-capstone`
- [ ] Tạo PVC hoặc config S3 CSI driver trong EKS
- [ ] Tạo SQS queue (signal queue + DLQ)
- [ ] Config IRSA cho `cdo-executor-sa` với đúng SQS/S3 permissions

### Phase 2 — Build preprocessor (T2-T3 W12)
- [ ] Implement Metrics Reader: đọc metrics.csv, tính delta Signal 1, emit Signal 2/3
- [ ] Implement Log Parser: lọc ERROR, PII redact, emit Signal 4
- [ ] Implement Trace Parser: lọc `statusCode != 0.0`, emit Signal 5
- [ ] Implement Tenant ID Injector
- [ ] Implement Schema Validator + DLQ routing

### Phase 3 — Deploy & Test (T3-T4 W12)
- [ ] Deploy `telemetry-preprocessor` vào namespace `platform`
- [ ] Verify SQS messages có đúng schema (spot check 10+ messages)
- [ ] Verify `tenant_id` được inject đúng theo dataset source
- [ ] Verify PII không xuất hiện trong SQS messages
- [ ] Verify DLQ nhận malformed payloads

### Phase 4 — Integration (T4-T5 W12)
- [ ] Confirm AI team đã connect SQS queue
- [ ] Chạy end-to-end: dataset → SQS → AI detect → AI decide → CDO execute
- [ ] Capture evidence: CloudWatch logs, SQS metrics, AI response

---

## 10. Open Questions cần chốt với AI team

| # | Câu hỏi | Ảnh hưởng |
|---|---|---|
| Q1 | RE2 dùng cho signal nào, RE3 dùng cho signal nào? (xem Bảng 6 — cần confirm) | Tenant ID inject sai thì AI sẽ không correlate đúng |
| Q2 | SQS Queue ARN là gì? 1 queue chung hay queue riêng từng signal? | CDO cần để config preprocessor |
| Q3 | AI có validate `dataset_source` label trong payload không? | CDO có thể gắn thêm metadata để debug |
| Q4 | Replay tốc độ dataset: CDO emit theo timestamp thật trong file hay theo real-time clock? | Ảnh hưởng AI sliding window detection |
| Q5 | AI skeleton endpoint available từ T mấy W12? | CDO cần để test integration |

---

## Related Documents

- `01_requirements_analysis.md` — requirements và pattern scope
- `02_infra_design.md` — kiến trúc tổng thể
- `03_security_design.md` — PII filter, RBAC, audit
- `telemetry-contract-review.md` — review chi tiết Telemetry Contract từ AI team
