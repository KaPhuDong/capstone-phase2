# Task Force 3 - Self-Heal Engine: CDO Deep Analysis
> Phân tích chi tiết TF3 dưới góc nhìn CDO: infra challenge, angle options, differentiation.

---

## Bức tranh tổng thể TF3

**Client**: VP Engineering · SaaS B2B · 200+ microservice · EKS multi-AZ single region us-east-1
**Pain**: 80% on-call page là known patterns → engineer thức 2h sáng click "restart" → burnout → eNPS 42→11 → retention -30% YoY
**Solution**: Self-Heal Engine tự động pipeline: `detect → match runbook → execute → verify → escalate`

### Infra Flow CDO phải build

```
[Prometheus AlertManager]
        |
        v (webhook)
[CDO Platform: alert ingestor]
        |
        v
[AI Engine: POST /v1/detect]  ──→  { anomaly_type, confidence, correlation_id }
        |
        v
[CDO: decision gate]
  ├── dry_run? → log only
  ├── blast_radius > limit? → escalate directly
  └── OK → proceed
        |
        v
[AI Engine: POST /v1/decide]  ──→  { action_plan[], blast_radius_ok, idempotency_key }
        |
        v
[CDO: execute via K8s API]
  (idempotency_key → DynamoDB conditional write = lock)
        |
        v
[AI Engine: POST /v1/verify]  ──→  { status: success|regression|next_action }
        |
   ┌────┴────┐
success   regression/fail
   |            |
[Audit log]  [Auto rollback → Escalate với context bundle → Slack webhook]
   |
[S3 Object Lock - tamper evident]
[Athena queryable]
```

---

## CDO Dimension Analysis

| Dimension | TF3 Specifics |
|-----------|--------------|
| **Data flow** | Bidirectional: alert webhook IN → K8s API OUT. Không phải request-response đơn thuần. Engine gọi K8s API để execute action. |
| **Compute pattern** | Near real-time response required. Pattern matching → action execution phải nhanh (detect → act < latency budget). |
| **Multi-tenant** | ≥2 tenants, RBAC isolation trong cluster. Namespace-per-tenant là minimum viable. |
| **Idempotency** | Critical: cùng incident không được execute 2 lần. DynamoDB conditional write hoặc Redis SETNX làm idempotency lock. |
| **Audit** | Tamper-evident bắt buộc (SOC2 Type II re-cert tháng 9). S3 Object Lock là minimum. Athena query interface. |
| **Safety** | 5 sub-checkpoint mandatory: dry-run, blast-radius, verify post-act, auto rollback, circuit breaker. |

---

## 3 Candidate Angles cho TF3 CDO

### Angle A: K8s Operator (EKS Native)

**Core bet**: CRD + controller = native K8s pattern, closest to production SRE tooling.

**Stack:**
```
EKS cluster
├── Custom CRD: SelfHealPolicy (defines patterns + rules)
├── Controller (Go/Python): watch alerts → reconcile state
├── Helm charts: deploy controller + AI engine
├── ArgoCD: GitOps, drift detection
├── Prometheus + Grafana: observability
├── External Secrets Operator: pull secrets từ AWS Secrets Manager
└── RBAC: ClusterRole per tenant namespace
```

**Win trên axis:**
- ✅ **Most production-realistic**: đây chính xác là pattern SRE teams dùng ngoài thực tế
- ✅ GitOps-native: ArgoCD drift detection + sync waves
- ✅ Namespace-per-tenant RBAC isolation: rõ ràng, K8s native
- ✅ Prometheus + Grafana: observability depth cao, phù hợp demo scenario

**Trade-off phải honest:**
- ⚠️ Highest build complexity: CRD + controller = ~2 ngày setup thay vì ~4 giờ
- ⚠️ EKS base cost: ~$0.10/hour control plane + node cost
- ⚠️ Go/Python controller từ scratch hoặc dùng kubebuilder → learning curve nếu team chưa làm
- ⚠️ Most moving parts: EKS + ArgoCD + Prometheus + ExternalSecrets = nhiều thứ có thể fail

**Differentiation story:**
> "Tôi chọn K8s operator vì TF3 về bản chất là K8s remediation - CDO kia dùng Step Functions để orchestrate nhưng không có drift detection hay GitOps. ArgoCD của tôi tự phát hiện config drift và auto-sync, tức là môi trường sandbox luôn ở trạng thái desired. Prometheus + Grafana cho observability native với K8s metrics (pod OOM events, restart counts) mà CDO kia không có. Trade-off: higher build cost $X vs $Y, mitigate bằng spot nodes."

---

### Angle B: Workflow Orchestration (Step Functions + EKS)

**Core bet**: Step Functions cho clear audit trail + visual debugging, EKS chỉ để run workload và nhận K8s actions.

**Stack:**
```
EKS cluster (workload sandbox only)
Step Functions: workflow engine
├── State: detect → decide → execute → verify → (rollback?)
├── Each step = Lambda calling AI endpoint
├── Built-in retry + error handling
├── Execution history = audit trail (+ S3 export)
DynamoDB: idempotency lock + state store
S3 Object Lock: final audit archive
CloudWatch: metrics + alarms
```

**Win trên axis:**
- ✅ Clear audit trail visual: Step Functions execution graph = intuitive cho demo
- ✅ Built-in retry/error handling: Lambda retry policy, catch states
- ✅ Faster to build base infra (~1 ngày vs ~2 ngày K8s operator)
- ✅ Lower cost: no persistent K8s operator pod
- ✅ Easier debugging: Step Functions console hiện rõ step nào fail

**Trade-off:**
- ⚠️ Less "native K8s": execution plane tách khỏi K8s cluster
- ⚠️ Step Functions cost tăng theo execution count (cần monitor)
- ⚠️ Latency: Step Functions orchestration overhead ~100-500ms per state transition
- ⚠️ Cold start nếu Lambda không warm

**Differentiation story:**
> "K8s operator CDO kia phức tạp hơn cần thiết cho capstone scope. Step Functions cho tôi visual execution graph mà mentor có thể xem real-time trong demo - mỗi step detect/decide/verify rõ ràng. Audit trail built-in qua execution history, export sang S3 Object Lock. Faster build = more time cho 5 safety checkpoints và E2E scenarios. Trade-off là less K8s-native feel."

---

### Angle C: Event-Driven Serverless (EventBridge + Lambda + EKS)

**Core bet**: EventBridge routing + Lambda functions = lowest cost, fastest initial build, fully serverless control plane.

**Stack:**
```
EKS cluster (workload sandbox only)
API Gateway: ingest alert webhook
Lambda (detect): call /v1/detect, route to EventBridge
EventBridge: event bus + rules routing
Lambda (decide): call /v1/decide, write DynamoDB lock
Lambda (execute): call K8s API (via kubeconfig in Secrets Manager)
Lambda (verify): call /v1/verify, write audit
S3 Object Lock: audit storage
DynamoDB: idempotency + state
```

**Win trên axis:**
- ✅ Lowest cost (pay-per-invocation)
- ✅ Fastest time-to-basic-infra (~4 giờ)
- ✅ EventBridge rules = flexible routing (easy thêm pattern mới)
- ✅ Decoupled: mỗi Lambda độc lập, dễ test riêng

**Trade-off:**
- ⚠️ Cold start risk: detect → decide → execute chain = multiple Lambda cold starts
- ⚠️ Less production-realistic cho K8s healing context
- ⚠️ Lambda 15-min limit: nếu verify loop lâu phải handle
- ⚠️ EventBridge ordering: không guarantee strict order nếu concurrent incidents

**Differentiation story:**
> "Tôi chọn serverless-first vì TF3 pattern execution frequency thực tế thấp (2-4 incidents/night). Pay-per-invocation = zero cost khi không có incident. Cost/tenant/month tôi ~$X, so với EKS operator always-on ~$Y. EventBridge rules cho phép thêm pattern mới không cần redeploy - chỉ thêm rule. Trade-off là cold start latency, mitigate bằng provisioned concurrency cho Lambda detect."

---

## Recommendation: Chọn Angle nào?

**Nếu CDO kia chưa announce angle:**
→ Chọn **Angle A (K8s Operator)** - strongest story cho TF3, highest rubric score potential cho infrastructure quality.

**Nếu CDO kia đã chọn K8s Operator:**
→ Chọn **Angle B (Step Functions)** - differentiated rõ ràng, faster build, clear audit story.

**Nếu team không strong K8s nhưng strong serverless:**
→ Chọn **Angle C** nếu CDO kia chọn K8s, hoặc **Angle B** nếu muốn middle ground.

**Không nên**: cả 2 CDO cùng K8s Operator → không differentiated → cap Tier 2.

---

## Infra Components CDO TF3 phải build (bất kể angle)

### Must-have (non-negotiable)

| Component | Purpose | AWS Service |
|-----------|---------|-------------|
| EKS sandbox cluster | Workload target + K8s API actions | EKS (1.29+) |
| Alert ingestor | Receive Prometheus AlertManager webhook | API GW + Lambda / ALB |
| AI engine integration | 3 endpoint calls (detect/decide/verify) | Lambda / Fargate / EKS pod |
| DynamoDB idempotency lock | Conditional write = prevent double-execute | DynamoDB |
| S3 Object Lock bucket | Tamper-evident audit storage | S3 (Object Lock enabled) |
| Athena + Glue | Query audit logs | Athena + Glue crawler |
| IAM ServiceAccount RBAC | K8s API least-privilege access | IAM + K8s RBAC |
| Secrets Manager | kubeconfig, AI engine creds | Secrets Manager |
| Namespace-per-tenant | RBAC isolation ≥2 tenants | K8s Namespace + RBAC |
| Slack webhook mock | Escalation notification target | Slack webhook / mock endpoint |

### Nice-to-have (differentiation)

| Component | Angle fit | Value |
|-----------|-----------|-------|
| ArgoCD | K8s Operator | GitOps + drift detection |
| Prometheus + Grafana | K8s Operator | Rich observability |
| Step Functions visual | Workflow | Demo-friendly audit trail |
| KEDA | K8s Operator | Event-based autoscaling |
| Circuit breaker (Hystrix-style) | All | 5th safety checkpoint |

---

## Key Terraform Modules cần build T6 W11

```
infra/
├── modules/
│   ├── networking/         # VPC, private/public subnets, SG, VPC endpoints
│   ├── eks-cluster/        # EKS cluster, node group, OIDC provider
│   ├── eks-rbac/           # ServiceAccount, ClusterRole, RoleBinding per tenant
│   ├── ai-engine/          # Compute hosting AI engine (Fargate/EKS pod)
│   ├── audit-storage/      # S3 Object Lock bucket, Glue crawler, Athena workgroup
│   ├── idempotency-lock/   # DynamoDB table (conditional write)
│   ├── alert-ingestor/     # API GW + Lambda / ALB ingestor
│   ├── observability/      # CloudWatch / Prometheus + Grafana
│   ├── secrets/            # Secrets Manager entries, rotation policy
│   └── tenant/             # Per-tenant namespace + RBAC (reusable module)
├── environments/
│   └── sandbox/
└── README.md
```

---

## 5 Safety Checkpoints - CDO phải implement tất cả

| Checkpoint | Implement ở đâu trong CDO flow |
|------------|-------------------------------|
| **1. Dry-run** | CDO check `dry_run_mode` từ request → nếu true, call `/v1/decide` nhưng KHÔNG execute K8s action, chỉ log |
| **2. Blast-radius** | CDO check `/v1/decide` response `blast_radius_ok` → nếu false, bypass execute → escalate thẳng |
| **3. Verify post-act** | CDO call `/v1/verify` sau execute, check `status` → nếu `regression` → trigger rollback |
| **4. Auto rollback** | CDO keep pre-state snapshot (hoặc Git SHA), nếu verify = regression → apply rollback |
| **5. Circuit breaker** | CDO track consecutive failures per pattern → sau N fails → open circuit, skip execute, escalate |

---

## 3 Known Patterns + 2 Designed-only (Hard requirement)

**3 patterns implement + test trên sandbox:**
1. **Pod OOMKilled** → adjust memory limit (patch deployment resources)
2. **Service stuck / CrashLoopBackOff** → restart deployment (rollout restart)
3. **Queue backlog** → scale worker (scale deployment replicas)

**2 patterns design-only (paper playbook + diagram + ADR):**
4. **Cert expiring** → rotate secret (design: trigger cert-manager rotation)
5. **Node pressure / DiskPressure** → drain node (design: cordon + drain + replace)

> Chọn 3 patterns implement theo business impact với Client. Ghi rõ lý do trong ADR.

---

## Scenario Simulation Plan (≥4h window, ≥10 scenarios)

CDO phải build test harness inject incidents vào sandbox cluster:

| # | Scenario | Pattern | Expected outcome |
|---|----------|---------|-----------------|
| 1 | OOM pod single instance | OOMKilled | Memory limit patched, pod restart OK |
| 2 | OOM pod persistent (3 times) | OOMKilled | After 3rd attempt → escalate |
| 3 | CrashLoopBackOff basic | Service stuck | Rollout restart, pod healthy |
| 4 | CrashLoopBackOff persistent | Service stuck | Circuit breaker open → escalate |
| 5 | Queue backlog 5x threshold | Queue backlog | Worker scaled up, backlog cleared |
| 6 | Queue backlog below threshold | Queue backlog | Blast radius check pass/fail |
| 7 | Dry-run mode ON | OOMKilled | Log only, no K8s action |
| 8 | Blast radius exceeded | Queue backlog | Escalate directly, no action |
| 9 | Verify: regression detected | Any | Auto rollback triggered |
| 10 | Multi-tenant: Tenant A incident | OOMKilled | Tenant B namespace untouched |
| 11 | Unknown pattern (low confidence) | N/A | Engine returns low confidence → escalate |
| 12 | Concurrent incidents × 2 | OOM + CrashLoop | Idempotency lock prevents double-execute |

**Test window**: inject theo schedule trong 4h, record `auto_resolved_count / total_scenarios` ≥ 60%.
