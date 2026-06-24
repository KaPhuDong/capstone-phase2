# CDO-02 Build Guide — Các đầu việc cần làm
**Doc owner:** CDO-02  
**Trạng thái:** Draft — W11/W12  
**Cập nhật lần cuối:** 2026-06-24  
**Mục đích:** Làm rõ CDO-02 cần build những gì, dùng công nghệ gì, tại sao, và các phần liên hệ nhau ra sao.  
**Môi trường:** Local = minikube (phát triển/test), Production demo = AWS EKS thật.

---

## 0. Bức tranh tổng thể — CDO-02 build gì?

CDO-02 build **5 nhóm thành phần** liên kết với nhau thành một pipeline khép kín:

```
┌─────────────────────────────────────────────────────────────────┐
│                        EKS / minikube                           │
│                                                                 │
│  [1. Workload Layer]        [2. Observability Layer]            │
│  Online Boutique pods   →   Prometheus + Fluent Bit + OTel     │
│  tenant-a / tenant-b        (thu metrics, logs, traces)         │
│           │                          │                          │
│           └──────────────────────────┘                          │
│                          │                                      │
│               [3. Telemetry Collector]                          │
│               CDO service: normalize,                           │
│               inject tenant_id, build                           │
│               signal_window → buffer                            │
│                          │                                      │
│               [4. Self-Heal Executor]                           │
│               Alert trigger → gọi AI API                        │
│               Safety Gate → dry-run                             │
│               Execute K8s action → verify                       │
│               Audit → escalate                                  │
└─────────────────────────────────────────────────────────────────┘
                           │  HTTPS + IAM SigV4
                           ▼
                    AI Engine (ECS Fargate)
                    /v1/detect  /v1/decide  /v1/verify

[5. IaC / Infra Layer]
Terraform: VPC, EKS, S3, IAM, SG
Helm/manifests: namespaces, RBAC, workloads
```

---

## 1. Workload Layer — Online Boutique trên Kubernetes

### Mục đích
Đây là **hệ thống sinh ra dữ liệu thật** — metrics, logs, traces. Không có workload này, không có gì để collect, không có gì để self-heal.

### CDO phải làm gì?

**1.1 Tạo 3 namespaces:**
```yaml
# manifests/namespaces/
platform.yaml    # chạy CDO executor, telemetry collector, OTel
tenant-a.yaml    # Online Boutique instance cho tenant A
tenant-b.yaml    # Online Boutique instance cho tenant B
```

**1.2 Deploy Online Boutique vào mỗi tenant namespace:**
- Source: https://github.com/GoogleCloudPlatform/microservices-demo
- Deploy bằng Helm hoặc kustomize, override namespace
- Gắn label `tenant_id` vào mọi pod/deployment

**1.3 Cấu hình Istio sidecar injection:**
```bash
kubectl label namespace tenant-a istio-injection=enabled
kubectl label namespace tenant-b istio-injection=enabled
```

### Tại sao cần Istio?
Signals 1 (`istio_request_error_rate`) và 2 (`istio_request_latency_p95`) trong Telemetry Contract đều đến từ Istio sidecar proxy. Không có Istio → không có 2 signals quan trọng nhất.

### Môi trường
| Giai đoạn | Môi trường | Ghi chú |
|---|---|---|
| Phát triển / test | minikube | Cần enable Istio addon hoặc install Istio manually |
| Demo với mentor | AWS EKS | Istio cài qua Helm chart `istio/base` + `istio/istiod` |

### Liên hệ với phần khác
→ Output của phần này là **dữ liệu thật** cho Observability Layer collect.

---

## 2. Observability Layer — Thu metrics, logs, traces

### Mục đích
Lớp này **chuyển hoạt động của workload thành data có cấu trúc** để Telemetry Collector xử lý tiếp. Gồm 3 stack riêng biệt cho 3 loại data.

---

### 2.1 Metrics — Prometheus + kube-prometheus-stack

**Thu gì:** `istio_requests_total`, `istio_request_duration_milliseconds`, `container_memory_working_set_bytes`

**Cách hoạt động:**
```
Istio sidecar trong pod → expose /metrics endpoint
Prometheus scrape mỗi 15s → lưu vào time-series DB
CDO Telemetry Collector → query Prometheus HTTP API → lấy giá trị
```

**Deploy:**
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace platform
```

**Queries CDO sẽ dùng:**
```promql
# Signal 1 — error rate
rate(istio_requests_total{response_code!~"2..",namespace="tenant-a"}[5m])
/
rate(istio_requests_total{namespace="tenant-a"}[5m])

# Signal 2 — latency p95
histogram_quantile(0.95,
  rate(istio_request_duration_milliseconds_bucket{namespace="tenant-a"}[5m])
)

# Signal 3 — memory
container_memory_working_set_bytes{namespace="tenant-a", container!="POD"}
```

---

### 2.2 Logs — Fluent Bit → CloudWatch Logs

**Thu gì:** Pod logs của Online Boutique services, lọc ERROR / Exception / stack trace

**Cách hoạt động:**
```
Pod stdout/stderr
→ Fluent Bit DaemonSet (chạy trên mỗi node) đọc log file
→ Parse, filter ERROR
→ Gửi lên CloudWatch Logs group: /cdo-02/eks/application
→ CloudWatch Logs Subscription Filter → trigger Lambda
→ Lambda normalize → gửi vào internal buffer của CDO Executor
```

**Deploy Fluent Bit:**
```bash
helm repo add aws https://aws.github.io/eks-charts
helm install aws-for-fluent-bit aws/aws-for-fluent-bit \
  --namespace platform \
  --set cloudWatch.enabled=true \
  --set cloudWatch.region=us-east-1 \
  --set cloudWatch.logGroupName=/cdo-02/eks/application
```

**Với minikube (local):** Fluent Bit gửi log ra file local hoặc stdout, CDO collector đọc trực tiếp từ Kubernetes API (`kubectl logs`) thay vì CloudWatch.

---

### 2.3 Traces — OpenTelemetry Collector → X-Ray / Jaeger

**Thu gì:** Trace spans có `status.code = ERROR` từ các service calls giữa Online Boutique services

**Cách hoạt động:**
```
Online Boutique services (có OTel SDK) → gửi spans → OTel Collector
OTel Collector → filter span ERROR → export sang X-Ray (AWS) hoặc Jaeger (local)
OTel Collector → đồng thời push error spans → internal buffer của CDO
```

**Deploy OTel Collector:**
```yaml
# manifests/platform/otel-collector.yaml
# Dùng OpenTelemetry Operator hoặc deploy Collector deployment trực tiếp
# Config: receivers: otlp; processors: filter(status=ERROR); exporters: awsxray + otlphttp(CDO)
```

**Lưu ý:** Online Boutique có sẵn một số services đã instrument OTel (Go services). Services Java/Python cần thêm auto-instrumentation agent. Đây là điểm cần verify khi deploy thực tế.

**Với minikube:** Export sang Jaeger thay vì X-Ray (Jaeger chạy local, không cần AWS).

### Liên hệ với phần khác
→ Output: Prometheus endpoint (`:9090`), CloudWatch Logs group, OTel error spans  
→ Tất cả được **Telemetry Collector** đọc và normalize tiếp theo.

---

## 3. Telemetry Collector — Normalize và chuẩn bị data cho AI

### Mục đích
Đây là **CDO-owned service**, đóng vai trò trung gian: đọc data thô từ Prometheus/CloudWatch/OTel, chuẩn hóa thành đúng schema Telemetry Contract, inject `tenant_id`, rồi giữ trong buffer để Executor lấy khi cần gọi AI.

### Tại sao cần component riêng?
AI API `/v1/detect` nhận `signal_window[]` — một mảng datapoints có cấu trúc chuẩn. Data thô từ Prometheus query trả về format khác, CloudWatch trả về format khác, OTel lại khác nữa. Telemetry Collector là nơi **normalize tất cả về 1 schema duy nhất** trước khi gửi AI.

### Về SQS — làm rõ luồng

Sau khi đọc kỹ contract và AI API, luồng thực tế như sau:

```
KHÔNG phải: CDO push signals liên tục vào SQS → AI tự poll

ĐÚNG là:
CDO Telemetry Collector → gom signals theo cửa sổ thời gian
                        → giữ trong internal buffer (in-memory hoặc Redis nhỏ)
                        → khi Executor cần gọi /v1/detect
                        → Executor lấy signal_window từ buffer
                        → nhét vào request body → POST /v1/detect
```

SQS trong Telemetry Contract template là thiết kế tham khảo cho trường hợp AI tự poll (async). Nhưng với AI API Contract của TF3 dùng synchronous REST (`/v1/detect`), CDO **không cần SQS**. CDO chỉ cần buffer nội bộ.

### Ngôn ngữ đề xuất: **Python**

- Dễ query Prometheus HTTP API (`requests` library)
- Boto3 cho CloudWatch Logs
- OpenTelemetry SDK cho traces
- Chạy gọn trong 1 Python pod trên EKS/minikube

### CDO phải làm gì?

**3.1 Metrics collector loop** (chạy mỗi 5 giây):
```
Query Prometheus API → lấy error_rate, latency_p95 cho từng service
Query Prometheus/CloudWatch → lấy memory_working_set cho từng pod
Normalize → đúng schema signal
Inject tenant_id từ namespace label
Lưu vào in-memory ring buffer (giữ 5 phút gần nhất)
```

**3.2 Log event listener** (real-time):
```
Subscribe CloudWatch Logs (production) hoặc poll kubectl logs (minikube)
Filter dòng có ERROR / Exception / Traceback
Parse → extract service, pod_name, level, message
Redact PII khỏi message (regex strip email, phone, token)
Push vào event queue
```

**3.3 Trace event listener** (real-time):
```
OTel Collector push error spans → Telemetry Collector HTTP endpoint
Hoặc: Telemetry Collector poll Jaeger API lấy error traces
Normalize → đúng schema trace_span_error_event
Push vào event queue
```

**3.4 Signal window builder:**
```
Khi Executor cần gọi AI:
→ Lấy N datapoints gần nhất từ ring buffer
→ Build signal_window[] đúng format AI API Contract
→ Trả về cho Executor
```

### Schema output (gửi vào signal_window):
```json
{
  "ts": "2026-06-25T10:30:00.123Z",
  "signal_name": "istio_request_error_rate",
  "value": 0.45,
  "tenant_id": "tenant-a",
  "labels": {
    "service": "adservice",
    "endpoint": "hipstershop.AdService/GetAds",
    "system": "OB"
  }
}
```

### Liên hệ với phần khác
→ Input: Prometheus, CloudWatch Logs, OTel Collector  
→ Output: `signal_window[]` buffer sẵn sàng cho Executor  
→ **Executor không query Prometheus trực tiếp** — mọi data đều qua Telemetry Collector

---

## 4. Self-Heal Executor — Trái tim của CDO platform

### Mục đích
Executor là **orchestrator chính**: nhận alert, lấy context từ Telemetry Collector, gọi AI API theo đúng thứ tự, enforce safety gate, execute action trên Kubernetes, verify kết quả, và ghi audit. Đây là component CDO phải build công phu nhất.

### Ngôn ngữ đề xuất: **Python**
- `kubernetes` Python client để gọi K8s API
- `boto3` cho AWS SigV4 signing khi gọi AI endpoint
- `httpx` hoặc `requests` cho HTTP calls
- Chạy như Deployment trong namespace `platform`

### Luồng chi tiết bên trong Executor

```
BƯỚC 1 — NHẬN ALERT / TRIGGER
────────────────────────────────
Trigger source (1 trong 2):
  A. Prometheus AlertManager → webhook POST vào Executor /ingest
  B. Executor tự poll metrics mỗi 30s → phát hiện threshold vượt ngưỡng

Khi nhận alert:
  → Tạo incident object: { correlation_id, tenant_id, namespace, service, ts }
  → Ghi audit: alert_received
  → Chuyển sang bước 2

BƯỚC 2 — GOM TELEMETRY CONTEXT
────────────────────────────────
  → Gọi Telemetry Collector: lấy signal_window[] (5 phút gần nhất)
  → Lấy thêm: Kubernetes events cho namespace (kubectl get events)
  → Lấy deployment version hiện tại
  → Ghi audit: telemetry_collected

BƯỚC 3 — GỌI AI /v1/detect
────────────────────────────────
  POST https://ai-engine.tf-3.internal/v1/detect
  Headers:
    Authorization: AWS SigV4
    X-Tenant-Id: tenant-a
    X-Correlation-Id: {correlation_id}
  Body:
    { signal_window: [...], context: { deployment_version, time_range } }

  Nếu AI trả anomaly: false → close incident, ghi audit
  Nếu AI trả 503 → KHÔNG execute, escalate + ghi audit
  Nếu AI trả anomaly: true → chuyển sang bước 4
  → Ghi audit: detect_called, detect_response

BƯỚC 4 — GỌI AI /v1/decide
────────────────────────────────
  POST /v1/decide (kèm context anomaly từ bước 3)
  AI trả về:
    { matched_runbook, action_plan[], blast_radius_config }

  → Ghi audit: decide_called, action_plan_received

BƯỚC 5 — SAFETY GATE (quan trọng nhất)
────────────────────────────────
  Kiểm tra lần lượt — BẤT KỲ check nào fail → DENY ngay:

  ✓ tenant_id trong action_plan == tenant_id của incident?
  ✓ target namespace có khớp tenant không?
  ✓ action_type có trong allow-list không?
       [RESTART_DEPLOYMENT, SCALE_UP_PODS, UPDATE_ENV_SECRET, ADJUST_MEMORY_LIMIT]
  ✓ blast_radius_config có trong giới hạn không? (max 1 deployment mỗi lần)
  ✓ rollback_plan có tồn tại không?
  ✓ verify_plan có tồn tại không?
  ✓ Idempotency-Key chưa từng dùng chưa? (check in-memory store)
  ✓ AI confidence >= threshold (cần chốt với AI team, tạm thời >= 0.7)

  Nếu DENY:
    → Ghi audit: safety_denied, reason
    → Escalate (Slack message / log) → STOP

  Nếu PASS:
    → Ghi audit: safety_passed
    → Chuyển sang bước 6

BƯỚC 6 — DRY-RUN
────────────────────────────────
  Gọi Kubernetes API với flag --dry-run=server
  Ví dụ RESTART_DEPLOYMENT:
    kubectl rollout restart deployment/{name} -n {namespace} --dry-run=server

  Nếu dry-run fail:
    → Ghi audit: dry_run_failed
    → Escalate → STOP

  Nếu dry-run pass:
    → Ghi audit: dry_run_passed
    → Chuyển sang bước 7

BƯỚC 7 — EXECUTE ACTION trên Kubernetes
────────────────────────────────
  Dùng kubernetes Python client, không dùng kubectl shell:

  RESTART_DEPLOYMENT:
    apps_v1.patch_namespaced_deployment(name, namespace,
      body với annotation restart timestamp)

  SCALE_UP_PODS:
    apps_v1.patch_namespaced_deployment_scale(name, namespace,
      body với replicas mới)

  ADJUST_MEMORY_LIMIT:
    apps_v1.patch_namespaced_deployment(name, namespace,
      body với resources.limits.memory mới)

  UPDATE_ENV_SECRET:
    core_v1.patch_namespaced_secret(name, namespace, body)

  → Ghi audit: execute_done, action_type, target, timestamp
  → Chờ action settle (wait for rollout, mặc định timeout 3 phút)

BƯỚC 8 — THU POST-ACTION TELEMETRY
────────────────────────────────
  → Chờ verify_plan.window_seconds (thường 60–180 giây)
  → Lấy signal_window mới từ Telemetry Collector
  → So sánh pre-action vs post-action signals

BƯỚC 9 — GỌI AI /v1/verify
────────────────────────────────
  POST /v1/verify
  Body: { action_taken: {...}, post_state: { signal_window: [...] } }

  AI trả về: { success, regression_detected, next_action }

  next_action = DONE     → close incident, ghi audit: incident_closed
  next_action = RETRY    → loop lại từ bước 3 (giới hạn 2 lần)
  next_action = ESCALATE → rollback + escalate

BƯỚC 10 — ROLLBACK (nếu cần)
────────────────────────────────
  Dùng rollback_plan từ action_plan:
    rollout_undo: kubectl rollout undo deployment/{name} -n {namespace}
  → Ghi audit: rollback_done
  → Escalate với context bundle đầy đủ

BƯỚC 11 — AUDIT (mọi bước đều ghi)
────────────────────────────────
  Mỗi event ghi vào S3 Object Lock bucket:
  {
    "correlation_id": "inc-001",
    "tenant_id": "tenant-a",
    "namespace": "tenant-a",
    "event": "safety_passed",
    "action_type": "RESTART_DEPLOYMENT",
    "decision": "execute",
    "result": "success",
    "reason": null,
    "timestamp": "2026-06-25T10:30:05.000Z"
  }
```

### Alert trigger — 2 cách CDO đề xuất

**Cách A (đề xuất cho production demo):** Prometheus AlertManager webhook
```
Prometheus rule: error_rate > 0.3 for 1m
→ AlertManager fires webhook → POST /ingest trên Executor
→ Executor nhận alert object với labels: namespace, service, severity
```

**Cách B (đề xuất cho dev/minikube):** Executor tự poll
```
Executor có background loop mỗi 30 giây
→ Query Telemetry Collector: có signal nào vượt ngưỡng không?
→ Nếu có → tự tạo incident → chạy pipeline
```

Cách B dễ implement hơn, không cần cấu hình AlertManager. Phù hợp cho giai đoạn build và test.

### Liên hệ với phần khác
→ Input: Telemetry Collector (signals), AI Engine (decisions), Kubernetes API (actions)  
→ Output: Kubernetes được heal, S3 audit trail, Slack escalation  
→ **Đây là component duy nhất có quyền gọi Kubernetes API để mutate workload**

---

## 5. IaC / Infra Layer — Terraform + Kubernetes manifests

### Mục đích
Tất cả infra phải được **code hóa** — không click tay trên console. Terraform cho AWS resources, Helm/YAML cho Kubernetes resources.

### 5.1 Terraform modules (target: AWS EKS production demo)

```
infra/
  envs/
    dev/          ← variables cho minikube/local (minimal)
    prod/         ← variables cho AWS EKS demo
  modules/
    vpc/          ← VPC, subnets, NAT gateway, security groups
    eks/          ← EKS cluster, node groups, IRSA config
    observability/← CloudWatch log groups, S3 audit bucket (Object Lock)
    iam/          ← IAM roles: executor role, deploy role, readonly role
```

**Resources quan trọng:**
- EKS cluster với node group đủ để chạy Online Boutique + CDO components
- S3 bucket với Object Lock Compliance Mode cho audit (retention 90 ngày)
- IAM role cho CDO executor pod (IRSA — không dùng static key)
- Security group: CDO executor SG được allow đến AI engine SG (port 8080)
- CloudWatch Log group: `/cdo-02/eks/application`

### 5.2 Kubernetes manifests

```
manifests/
  namespaces/
    platform.yaml       ← namespace platform
    tenant-a.yaml       ← namespace tenant-a + label istio-injection=enabled
    tenant-b.yaml       ← namespace tenant-b + label istio-injection=enabled
  rbac/
    executor-role.yaml  ← Role: get/list/watch/patch deployments trong tenant-*
    executor-rb.yaml    ← RoleBinding executor SA → Role
  platform/
    telemetry-collector.yaml  ← Deployment, Service, ConfigMap
    executor.yaml             ← Deployment, Service, ServiceAccount
    otel-collector.yaml       ← Deployment + ConfigMap
  workloads/
    online-boutique-tenant-a/ ← Helm values hoặc kustomize overlay
    online-boutique-tenant-b/
```

### 5.3 Với minikube (local development)

Các bước để chạy được local:
```bash
# 1. Start minikube với đủ resource
minikube start --cpus=4 --memory=8g --driver=docker

# 2. Enable addons
minikube addons enable metrics-server
minikube addons enable ingress

# 3. Install Istio
istioctl install --set profile=demo

# 4. Install Prometheus (kube-prometheus-stack)
helm install monitoring prometheus-community/kube-prometheus-stack -n platform

# 5. Deploy namespaces + RBAC
kubectl apply -f manifests/namespaces/
kubectl apply -f manifests/rbac/

# 6. Deploy Online Boutique vào tenant-a
kubectl apply -f manifests/workloads/online-boutique-tenant-a/ -n tenant-a

# 7. Deploy CDO components
kubectl apply -f manifests/platform/
```

Thay thế cho AWS khi dùng local:
| AWS Service | Local alternative |
|---|---|
| CloudWatch Logs | Loki hoặc đọc trực tiếp qua kubectl logs |
| S3 Object Lock | MinIO local hoặc ghi file JSON |
| AWS X-Ray | Jaeger (chạy trong minikube) |
| IAM SigV4 | Bearer token hoặc skip auth khi AI mock |
| SQS | Không cần (CDO dùng internal buffer) |

---

## 6. Mối liên hệ giữa 5 phần — dependency map

```
[5. IaC]
  └── tạo ra môi trường (cluster, namespaces, RBAC, S3, IAM)
        │
        ▼
[1. Workload Layer]
  └── Online Boutique chạy trong tenant-a, tenant-b
        │  sinh ra metrics, logs, traces
        ▼
[2. Observability Layer]
  Prometheus ──────────────────────────────────┐
  Fluent Bit → CloudWatch Logs ────────────────┤→ raw data
  OTel Collector → Jaeger/X-Ray ───────────────┘
        │
        ▼
[3. Telemetry Collector]  (CDO Python service)
  └── normalize, inject tenant_id, buffer signal_window[]
        │  signal_window[] sẵn sàng
        ▼
[4. Self-Heal Executor]  (CDO Python service)
  └── trigger → lấy context → gọi AI → safety → execute K8s → verify → audit
        │
        ├──→ AI Engine /v1/detect, /v1/decide, /v1/verify
        ├──→ Kubernetes API (execute action)
        └──→ S3 audit bucket
```

**Build order đề xuất:**
1. IaC skeleton (namespace, RBAC) — không cần AWS, chạy được trên minikube
2. Workload Layer — deploy Online Boutique, verify pods running
3. Observability Layer — verify Prometheus có metrics từ Istio sidecars
4. Telemetry Collector — viết Python service, verify đọc được Prometheus + logs
5. Self-Heal Executor — viết từng bước, test từng sub-component riêng

---

## 7. Các đầu việc cụ thể — tổng hợp

| # | Đầu việc | Output cần có | Phụ thuộc | Môi trường |
|---|---|---|---|---|
| 1 | Terraform module VPC + EKS | `terraform plan` không lỗi | — | AWS (prod) |
| 2 | Terraform module S3 Object Lock | Bucket tạo được với Object Lock | — | AWS (prod) |
| 3 | Terraform module IAM + IRSA | IAM role cho executor | EKS | AWS (prod) |
| 4 | minikube setup script | Cluster + Istio + Prometheus chạy | — | Local |
| 5 | Kubernetes manifests: namespaces + RBAC | 3 namespaces + Role/RoleBinding | minikube/EKS | Both |
| 6 | Deploy Online Boutique tenant-a | Pods running, Istio sidecar injected | #5, Istio | Both |
| 7 | Deploy Online Boutique tenant-b | Pods running, Istio sidecar injected | #5, Istio | Both |
| 8 | Verify Prometheus metrics | `istio_requests_total` có data | #6, #7 | Both |
| 9 | Setup Fluent Bit + CloudWatch Logs | Log group có pod logs | #6, #7 | AWS (prod) |
| 10 | Deploy OTel Collector | Error spans được collect | #6, #7 | Both |
| 11 | Viết Telemetry Collector service | API trả đúng signal_window[] | #8, #9, #10 | Both |
| 12 | Viết Executor — alert trigger | Nhận alert, tạo incident object | #11 | Both |
| 13 | Viết Executor — AI API client | Gọi /v1/detect, /v1/decide, /v1/verify | AI endpoint | Both |
| 14 | Viết Executor — Safety Gate | Deny cross-tenant test pass | #13 | Both |
| 15 | Viết Executor — K8s action handler | RESTART_DEPLOYMENT chạy được | #14, RBAC | Both |
| 16 | Viết Executor — audit writer | S3 object được ghi sau mỗi event | #15, S3 | Both |
| 17 | Viết Executor — verify + rollback | Verify gọi AI, rollback khi fail | #15, #16 | Both |
| 18 | Integration test: end-to-end scenario | 1 incident tự heal hoàn chỉnh | #11–#17 | Both |
| 19 | Multi-tenant isolation test | Cross-tenant action bị deny | #14 | Both |

---

## 8. Câu hỏi còn mở — cần chốt trước khi build

Các điểm dưới đây ảnh hưởng trực tiếp đến implementation, cần có câu trả lời trước W12:

| # | Câu hỏi | Ảnh hưởng đến |
|---|---|---|
| A | AI endpoint mock khi nào có? | Executor không test được nếu chưa có endpoint |
| B | `tenant_id` format AI accept: `"tenant-a"` hay format khác? | Telemetry Collector inject đúng hay không |
| C | Confidence threshold để execute là bao nhiêu? | Safety Gate logic |
| D | Retry policy: Executor retry mấy lần trước khi escalate? | Executor loop logic |
| E | Escalation channel: Slack webhook? Email? Log only? | Executor escalate step |
| F | Istio có bắt buộc không, hay có thể dùng native K8s metrics thay thế? | Signals 1 & 2 có data không |
| G | Trainer confirm EKS thật bắt buộc ở demo hay minikube được chấp nhận? | IaC scope và cost |
