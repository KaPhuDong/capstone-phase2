# AI Contracts Overview — Giải thích 3 Contracts
**Dành cho:** CDO-02 team  
**Nguồn:** `templates/ai/contracts/`  
**Mục đích:** Đọc nhanh, hiểu rõ 3 contracts là gì, để làm gì, và chúng liên quan nhau như thế nào  

---

## Big picture — 1 câu mỗi contract

| Contract | Trả lời câu hỏi | Ai viết | Ai đọc |
|---|---|---|---|
| **Telemetry Contract** | CDO phải gửi **data gì** cho AI? | AI team | CDO team |
| **AI API Contract** | CDO phải **gọi AI** như thế nào? | AI team | CDO team |
| **Deployment Contract** | AI engine **chạy ở đâu**, config ra sao? | AI team | CDO team |

> 3 contracts này do **AI team viết và giữ bản gốc**. CDO team đọc và implement theo.  
> Tất cả đều có trạng thái `🔒 FREEZE` — muốn thay đổi phải mở formal change request.

---

## Hình dung mối quan hệ

```
CDO Platform
    │
    │  ① Telemetry Contract
    │  CDO gom signals từ infra/dataset
    │  → gửi qua SQS
    ▼
AI Engine (ECS Fargate)        ← ③ Deployment Contract
    │                               mô tả AI engine deploy ở đâu
    │  ② AI API Contract            CDO cần biết để kết nối
    │  CDO gọi POST /v1/detect
    │  CDO gọi POST /v1/verify
    ▼
CDO nhận kết quả
→ Safety Gate → Execute → Audit
```

**Luồng thực tế:**
1. CDO dùng **Telemetry Contract** để biết cần thu thập và chuẩn hóa data nào.
2. CDO dùng **Deployment Contract** để biết AI endpoint ở đâu, auth ra sao, connect thế nào.
3. CDO dùng **AI API Contract** để gọi đúng API, gửi đúng schema, xử lý đúng response.

---

## Contract 1 — Telemetry Contract

### Là gì?
Danh sách **signals (tín hiệu đo lường)** mà CDO có trách nhiệm thu thập từ hệ thống và đẩy lên cho AI engine tiêu thụ.

### CDO phải làm gì?
- Đọc data từ metrics/logs/traces (dataset RE2/RE3 hoặc infra thật).
- Chuẩn hóa thành JSON đúng schema.
- Inject `tenant_id` vào mọi payload.
- Gửi qua **SQS** theo đúng tần suất và SLA.

### 5 signals trong contract TF3

| Signal | Loại | Tần suất | CDO làm gì |
|---|---|---|---|
| `istio_request_error_rate` | Gauge | 5 giây | **Tính toán** từ 2 counters: `Δerror / Δrequest` |
| `istio_request_latency_p95` | Gauge | 5 giây | Đọc thẳng từ `metrics.csv` |
| `container_memory_working_set_bytes` | Gauge | 10 giây | Đọc thẳng từ `metrics.csv` |
| `app_log_error_event` | Event | Real-time | Parse `logs.csv`, lọc ERROR + stack trace |
| `trace_span_error_event` | Event | Real-time | Parse `traces.csv`, lọc `statusCode != 0.0` |

### Keywords quan trọng
- **`tenant_id`** — bắt buộc trong mọi payload, CDO tự inject khi dùng offline dataset.
- **RFC3339 UTC** — định dạng timestamp bắt buộc, độ chính xác millisecond.
- **PII anonymization** — CDO phải lọc dữ liệu nhạy cảm trước khi gửi log sang AI.
- **Derived metric** — Signal 1 không đọc thẳng mà phải tính, đây là điểm phức tạp nhất.
- **Dead-letter queue** — AI sẽ reject payload sai schema, CDO cần xử lý DLQ.

---

## Contract 2 — AI API Contract

### Là gì?
Đặc tả **API endpoints** mà AI engine expose để CDO gọi. CDO dùng API này để hỏi AI "đây là lỗi gì?" và "action đó có thành công không?".

### 2 endpoints chính

#### `POST /v1/detect`
> **Hỏi AI:** "Dữ liệu này có bất thường không?"

- CDO gửi: `signal_window[]` (danh sách datapoints) + `context` (version, time range).
- AI trả về: `anomaly` (bool), `severity`, `suggested_action`, `confidence`, `audit_id`.
- **`confidence`** là trường quan trọng — CDO dùng để quyết định có execute action không (gating).

#### `POST /v1/verify`
> **Hỏi AI:** "Action vừa thực hiện có giải quyết được lỗi không?"

- CDO gửi: action đã execute + signals đo được sau action.
- AI trả về: `success` (bool), `regression_detected`, `next_action` (`DONE` / `RETRY` / `ESCALATE`).

### SLA API

| Endpoint | P99 Latency | Throughput | Availability |
|---|---|---|---|
| `/v1/detect` | < 500 ms | 100 RPS | 99.5% |
| `/v1/verify` | < 800 ms | — | 99.5% |

### Error codes CDO cần xử lý

| Code | Ý nghĩa | CDO phải làm gì |
|---|---|---|
| `400` | Schema sai | Sửa code, **không retry** |
| `401` | Auth fail | Refresh credential, retry 1 lần |
| `429` | Rate limit | Exponential backoff (1s → 2s → 4s...) |
| `503` | AI unavailable | **Bắt buộc** có fallback path (rule-based hoặc escalate) |

### Keywords quan trọng
- **`X-Tenant-Id`** — header bắt buộc trong mọi request.
- **`X-Correlation-Id`** — header trace end-to-end workflow (optional nhưng nên có).
- **IAM SigV4** — cơ chế auth, không dùng API key.
- **`confidence` gating** — CDO dùng ngưỡng confidence để quyết định execute hay escalate.
- **Fallback path** — khi AI trả 503, CDO không được execute mặc định, phải escalate.

---

## Contract 3 — Deployment Contract

### Là gì?
Mô tả **AI engine được deploy như thế nào** — chạy trên compute nào, network ra sao, endpoint ở đâu, auth thế nào. CDO cần biết để config infra kết nối được.

### Điểm then chốt: AI chạy SHARED

> AI team deploy **1 instance duy nhất** cho cả task force. Tất cả CDO trong nhóm (CDO-A1, CDO-A2, CDO-A3) cùng trỏ đến **1 endpoint duy nhất**, phân biệt nhau bằng `tenant_id`.

```
CDO-02 ──┐
CDO-03 ──┼──► https://ai-engine.tf-<N>.internal/  (1 shared endpoint)
CDO-04 ──┘
```

### Thông tin CDO cần để kết nối

| Thông tin | Giá trị |
|---|---|
| **Endpoint** | `https://ai-engine.tf-<N>.internal/` (internal ALB, private subnet) |
| **Port** | 8080 |
| **Auth** | IAM SigV4 (không có API key) |
| **Health check** | `GET /health` |
| **Network** | CDO phải nằm trong cùng VPC task force, SG-to-SG reference |

### AI engine chạy trên gì?
- **Compute:** ECS Fargate (hoặc Lambda nếu load nhẹ).
- **Scale:** min 2, max 10 tasks — auto-scale theo CPU 70% hoặc 100 req/task.
- **Rollout:** Canary (10% → 50% → 100%), auto rollback nếu error rate > 1% hoặc P99 > 800ms.
- **Secrets:** không có long-lived key, mọi credential qua AWS Secrets Manager.

### Keywords quan trọng
- **Internal ALB** — AI không có public endpoint, CDO phải ở trong cùng VPC.
- **SG-to-SG ingress** — security group CDO platform phải được AI engine allow ingress.
- **Canary rollout** — AI có thể deploy version mới bất cứ lúc nào theo canary, CDO cần handle backward compatibility.
- **ArgoCD rollback** — target RTO < 60 giây khi rollback.
- **OpenTelemetry** — AI emit traces về Jaeger/X-Ray, CDO có thể tích hợp để trace end-to-end.

---

## Tương tác giữa 3 contracts trong 1 workflow

Lấy ví dụ: **adservice bị error rate spike** → self-heal chạy.

```
[Telemetry Contract]
CDO đọc metrics.csv
→ tính Δerror/Δrequest cho adservice
→ inject tenant_id = "tnt-re3-simulation"
→ gửi signal qua SQS
                    ↓
[Deployment Contract]             [AI API Contract]
CDO biết AI ở đâu          →     CDO gọi POST /v1/detect
(internal ALB, port 8080)         gửi signal_window + context
(auth: IAM SigV4)
                    ↓
                AI trả về:
                anomaly: true, confidence: 0.91, suggested_action: RESTART_DEPLOYMENT

[CDO Safety Gate]
→ kiểm tra confidence đủ không?
→ validate tenant/namespace/blast-radius
→ dry-run → execute RESTART_DEPLOYMENT

[Telemetry Contract]              [AI API Contract]
CDO thu post-action signals  →    CDO gọi POST /v1/verify
(latency giảm chưa?)              gửi action_taken + post_state signals
                    ↓
                AI trả về:
                success: true, next_action: DONE

CDO ghi audit → close incident
```

---

## Tóm tắt dependency giữa 3 contracts

```
Deployment Contract
    └── CDO cần biết endpoint/network/auth
        └── để gọi được API API Contract
                └── API Contract cần nhận đúng signal format
                        └── format định nghĩa trong Telemetry Contract
```

**Nếu một contract thay đổi:**

| Contract thay đổi | Ảnh hưởng |
|---|---|
| Telemetry Contract (thêm/đổi signal) | CDO phải cập nhật data pipeline + schema validation |
| AI API Contract (thêm field, đổi SLA) | CDO phải cập nhật request/response handling |
| Deployment Contract (đổi endpoint, auth) | CDO phải cập nhật network config, IAM role, env vars |

---

## Checklist CDO cần implement từ 3 contracts

- [ ] **Telemetry:** Pipeline đọc RE2/RE3 dataset → chuẩn hóa → inject `tenant_id` → push SQS
- [ ] **Telemetry:** Xử lý delta calculation cho `istio_request_error_rate`
- [ ] **Telemetry:** PII filter trước khi gửi log events
- [ ] **API:** Implement gọi `/v1/detect` với đúng headers + schema
- [ ] **API:** Implement gọi `/v1/verify` sau khi execute action
- [ ] **API:** Implement fallback path khi AI trả `503`
- [ ] **API:** Implement exponential backoff khi gặp `429`
- [ ] **Deployment:** Config network/SG để CDO platform connect được internal ALB
- [ ] **Deployment:** Setup IAM role với SigV4 signing
- [ ] **Deployment:** Gắn `X-Tenant-Id` và `X-Correlation-Id` vào mọi request
