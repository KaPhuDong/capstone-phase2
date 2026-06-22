# Differentiation Angles - TF3 Self-Heal Engine
> Phân tích 3 angle chính cho TF3 CDO, cách xây dựng story vượt trội, và nguyên tắc chọn angle.
> Đọc trước T3 W11 khi lock angle.

---

## Nguyên tắc chọn angle (nhắc lại)

1. **Buildable trong 6 ngày W12** - không "ideal production blueprint"
2. **Differentiated rõ ràng** - khác CDO kia trong TF ít nhất 2 axis
3. **Story-able** - có thể nói "infra của tôi tốt hơn ở X, evidence là Y"
4. **Chọn theo skill team** - team strong K8s → K8s angle; team strong serverless → Step Functions/Serverless

---

## 3 Angles cho TF3

### Angle A: K8s Operator (EKS Native) ⭐ Recommended nếu team có K8s skill

**Core bet**: CRD + controller = native K8s pattern, closest to production SRE tooling. Đây là cách SRE teams thật sự build self-healing system ngoài thực tế.

**Full stack:**
```
EKS cluster (1.29+)
├── Custom CRD: SelfHealPolicy
│   └── Defines: pattern_name, match_condition, action, blast_radius_limit, retry_count
├── Controller (Python/Go với kopf/kubebuilder)
│   └── Watches AlertManager webhook → calls AI engine → executes action
├── Helm charts (deploy controller + dependencies)
├── ArgoCD (GitOps)
│   ├── Sync waves: CRDs → Controller → Workloads
│   ├── Drift detection (auto-sync disabled cho safety)
│   └── Application health checks
├── Prometheus + Grafana
│   ├── Custom metrics: incidents_resolved_total, circuit_breaker_open
│   ├── Dashboards: Self-Heal activity, pattern breakdown
│   └── Alerts: circuit breaker open → Slack
├── External Secrets Operator → AWS Secrets Manager
├── IRSA (IAM Roles for Service Accounts) cho AI engine
└── Namespace isolation: tenant-a, tenant-b (separate RBAC)
```

**Win axis vs other angles:**
| Axis | K8s Operator | Step Functions | Serverless |
|------|-------------|----------------|-----------|
| Production realism | ⭐⭐⭐ Highest | ⭐⭐ Medium | ⭐ Lower |
| GitOps native | ✅ ArgoCD built-in | ❌ Separate | ❌ Separate |
| Observability depth | ✅ Prometheus native | CloudWatch only | CloudWatch only |
| Build time | ~2 ngày | ~1 ngày | ~0.5 ngày |
| Risk if something breaks | High | Medium | Low |
| Cost (EKS always-on) | $X higher | $Y lower | $Z lowest |

**Trade-off phải honest:**
- ⚠️ Highest build complexity: CRD schema + controller reconcile loop = 2 ngày setup
- ⚠️ EKS control plane: ~$0.10/hour → ~$4.8/day → ~$29/2 tuần (just control plane)
- ⚠️ More things can break: ArgoCD + Prometheus + ExternalSecrets + IRSA = 4 systems
- ⚠️ Nếu team chưa làm kopf/kubebuilder trước: steep learning curve

**When to choose Angle A:**
- Team có người đã viết K8s controller hoặc operator trước
- CDO kia đã announce Step Functions / Serverless angle
- Team muốn highest rubric score potential (production realism)

**Differentiation story vs Step Functions:**
> "K8s Operator là cách production SRE teams thật sự build self-healing. CDO kia dùng Step Functions - tốt cho audit trail nhưng execution plane tách khỏi K8s. Controller của tôi chạy IN CLUSTER, có direct access K8s events, ArgoCD phát hiện drift ngay nếu ai thay đổi SelfHealPolicy. Prometheus cho tôi custom metrics K8s-native mà CDO kia không có: `kube_pod_container_status_restarts_total` feed trực tiếp vào alert detection. Trade-off: cost cao hơn $X/tháng."

---

### Angle B: Workflow Orchestration (Step Functions + EKS) ⭐ Recommended nếu team mixed skill

**Core bet**: Step Functions cho clear, debuggable, auditable workflow. EKS chỉ làm workload target + K8s API executor.

**Full stack:**
```
EKS cluster (workload sandbox target only - không run self-heal engine)
Step Functions Express Workflow
├── State: Detect → Gate → Decide → Execute → Verify → (Rollback?)
├── Each state = Lambda function calling AI endpoint
├── Built-in retry (MaxAttempts, BackoffRate per state)
├── Catch states: escalate on failure
├── Execution history = primary audit trail
└── Express Workflow: high-volume, low cost

Lambda functions:
├── alert-ingestor: receive webhook → start execution
├── detect-fn: call /v1/detect, extract anomaly_type
├── gate-fn: check blast_radius, idempotency lock
├── decide-fn: call /v1/decide, extract action_plan
├── execute-fn: call K8s API (kubeconfig from Secrets Manager)
├── verify-fn: call /v1/verify, check status
└── rollback-fn: apply previous state

DynamoDB: idempotency lock + workflow state
S3 Object Lock: archive execution history (tamper-evident)
Athena + Glue: query archived audit
CloudWatch: metrics + alarms
```

**Win axis vs K8s Operator:**
| Axis | Step Functions | K8s Operator | Serverless |
|------|---------------|-------------|-----------|
| Audit trail visual | ⭐⭐⭐ Step Functions console | ⭐ K8s events only | ⭐ CloudWatch logs |
| Debug-friendly | ✅ Execution graph | Limited | Limited |
| Build speed | ✅ ~1 ngày | ~2 ngày | ~0.5 ngày |
| Cost (pay-per-exec) | $0.000001/state-transition | Higher (always-on) | Lowest |
| Retry logic | ✅ Built-in per state | Manual in controller | Manual in Lambda |
| Production realism | Medium | Highest | Lower |

**Trade-off:**
- ⚠️ Less "native K8s" - controller không chạy in-cluster
- ⚠️ Step Functions Express: max 5 phút execution → verify loop phải handle
- ⚠️ Lambda cold start: nếu incidents rare → cold start ảnh hưởng detect → act latency
- ⚠️ Step Functions latency: ~100-500ms per state transition overhead

**When to choose Angle B:**
- CDO kia đã announce K8s Operator angle
- Team không confident với K8s controller code
- Team muốn faster base infra build (more time for safety checkpoints + scenarios)
- Team muốn demo-friendly: Step Functions console hiện execution graph real-time

**Differentiation story vs K8s Operator:**
> "Step Functions cho tôi something K8s Operator không có: visual execution graph mà mentor có thể xem real-time trong buổi demo. Mỗi step detect/decide/execute/verify hiển thị status, retry count, error message ngay trên console. Audit trail của tôi là execution history export sang S3 Object Lock - không cần query Athena cho simple cases. Built-in retry per state = không cần viết retry logic custom. Cost lower vì pay-per-execution. Trade-off: less K8s-native, controller không run in-cluster."

---

### Angle C: Event-Driven Serverless (EventBridge + Lambda + EKS)

**Core bet**: Fully serverless control plane, lowest cost, fastest build. EKS chỉ là workload target.

**Full stack:**
```
API Gateway: receive AlertManager webhook
Lambda (detect): call /v1/detect → publish to EventBridge
EventBridge: rules routing by anomaly_type
  ├── Rule: OOMKilled → Lambda(oom-handler)
  ├── Rule: CrashLoop → Lambda(crash-handler)
  └── Rule: QueueBacklog → Lambda(queue-handler)
Lambda per pattern: check idempotency → call /v1/decide → execute K8s action
Lambda (verify): call /v1/verify → S3 audit write
DynamoDB: idempotency lock + state
S3 Object Lock: audit
Athena + Glue: query
```

**Win axis:**
- ✅ Lowest cost: ~$0 when no incidents
- ✅ Fastest base infra build (~4 giờ vs ~2 ngày K8s Operator)
- ✅ EventBridge rules = add new pattern without redeploy (ADR-worthy)
- ✅ Each Lambda = isolated, testable independently

**Trade-off:**
- ⚠️ Cold start chain: 3-4 Lambda cold starts in sequence = latency
- ⚠️ EventBridge ordering: concurrent incidents might interleave
- ⚠️ Less production-realistic for K8s healing context
- ⚠️ Weakest differentiation story if CDO kia also serverless

**When to choose Angle C:**
- CDO kia đã chọn BOTH K8s Operator AND Step Functions (khó xảy ra vì chỉ 2 CDO)
- Team chỉ strong Lambda/serverless, không có K8s hoặc Step Functions experience
- Generally: **avoid** nếu có lựa chọn tốt hơn

---

## Decision Matrix TF3

```
CDO kia chưa announce angle?
└── Team có K8s skill? → Angle A (K8s Operator)
└── Team không có K8s skill? → Angle B (Step Functions)

CDO kia announce K8s Operator?
└── → Angle B (Step Functions) ✅

CDO kia announce Step Functions?
└── Team có K8s skill? → Angle A (K8s Operator) ✅
└── Team không có K8s skill? → Angle C (Serverless) - acceptable

Cả 2 CDO trùng angle?
└── → Escalate ke lead, ai switch sang angle khác (T3 W11)
```

---

## 5 Safety Checkpoints - Story cho buổi chấm

**TF3 yêu cầu 5 safety sub-checkpoints mandatory.** Đây là điểm CDO PHẢI demo được, không phải chỉ document.

| Checkpoint | Demo cụ thể |
|------------|------------|
| **1. Dry-run** | Inject incident với `dry_run=true` → show CloudWatch/Grafana/Step Functions: AI được call, K8s action KHÔNG được execute |
| **2. Blast-radius** | Inject incident với large-scale pattern → show `/v1/decide` trả `blast_radius_ok: false` → CDO route sang escalate |
| **3. Verify post-act** | Inject scenario có regression → show `/v1/verify` trả `regression` → CDO trigger rollback automatically |
| **4. Auto rollback** | Chứng minh rollback hoạt động: show trước/sau state của K8s resource |
| **5. Circuit breaker** | Inject 3 consecutive failures cùng pattern → show circuit breaker open: subsequent incidents của pattern này → escalate direct |

**Story cho pitch:**
> "5 safety checkpoints không phải checkbox trên doc - tôi có thể demo từng cái ngay bây giờ. [Mở demo] Đây là dry_run mode - AI được call, không có gì thay đổi trong cluster. Đây là blast radius exceeded - CDO tự động escalate. Đây là circuit breaker open sau 3 failures..."

---

## Differentiation Metrics cần đo

| Metric | Source | Dùng để defend |
|--------|--------|----------------|
| Auto-resolve rate | scenario simulation log | "≥60% target, tôi đạt X%" |
| $/tenant/month | AWS Cost Explorer W12 | vs CDO kia's angle estimate |
| p99 latency detect+decide+execute | CloudWatch / Grafana | SLA evidence |
| Time to onboard new tenant | Measured T3 W12 | Ops axis |
| Audit query time (Athena) | Athena execution history | Audit usability |
| Circuit breaker open events | Custom metric | Safety depth evidence |

---

## Anti-patterns TF3

❌ **Không demo safety checkpoints** → Reviewer hỏi "dry-run implement thế nào?" không trả lời được → cap Tier 2

❌ **Audit log không queryable** → S3 Object Lock có nhưng không có Athena → "tamper-evident" nhưng không ai query được → weak evidence

❌ **RBAC quá rộng** → ServiceAccount có `cluster-admin` → "zero unsafe action" fail → critical rubric miss

❌ **Idempotency không test** → Demo inject cùng incident 2 lần → double-execute → embarrassing

❌ **Cả 2 CDO cùng angle** → Differentiation story collapse → cả 2 cap Tier 2
