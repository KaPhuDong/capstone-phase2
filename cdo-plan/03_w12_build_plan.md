# W12 Build Plan - CDO Team · TF3 Self-Heal Engine
> Tuần 29/06 - 03/07. Mục tiêu: Full platform + AI integration + scenario simulation ≥4h + chaos test + code freeze 8h T5.

---

## Tổng quan tuần

```
T2 29/06      T3 30/06        T4 01/07           T5 02/07
[CI/CD]       [E2E test]      [Polish + Chaos]   [8h FREEZE]
[Observ.]     [Integration    [Curveball #3]     [Pitch onsite]
[AI integrate] session]       [Dry-run]
               [Curveball #2]
```

**Cột mốc cứng:**
- T2 16h: Curveball #2 medium (30p) → respond + ADR update
- T3 14h-16h: Integration session - gọi AI endpoint THẬT
- T4 14h: Curveball #3 chaos (60p) → respond + infra fix
- T4 18h: Code freeze + git tag `final` + Pack #2 submit
- T5 8h: Code freeze hard (sau giờ này chỉ sửa slides)
- T5 onsite: Pitch + individual defense

---

## T2 29/06 - CI/CD + Observability + AI Integration

### Mục tiêu ngày

1. CI/CD pipeline chạy được (build → test → deploy)
2. Observability stack live (metrics, logs, traces)
3. AI skeleton endpoint integrated vào platform flow

### Sáng: CI/CD Pipeline

**Phân công:**

| Track | Task | Done when |
|-------|------|-----------|
| CI/CD lead | GitHub Actions: build → test → scan → plan | Pipeline green trên PR |
| Infra | Terraform canary deploy module | Canary shift 10% works |
| Security | Gitleaks scan trong CI, OIDC setup | No secret leak detected |
| Ops | CloudWatch dashboards + alarms | Metrics live on dashboard |

**CI/CD minimum viable:**
```yaml
# .github/workflows/deploy.yml - gợi ý structure
on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  build-test-scan:
    steps:
      - checkout
      - build (docker build / terraform init+validate)
      - test (unit + integration)
      - scan (trivy, gitleaks)
      - plan (terraform plan)
  
  deploy-sandbox:  # on merge to develop
    needs: build-test-scan
    steps:
      - terraform apply -target=module.sandbox
      - smoke-test

  deploy-canary:   # on merge to main
    needs: build-test-scan
    steps:
      - terraform apply (10% canary)
      - wait + check error rate
      - promote or rollback
```

**Observability stack (pick theo angle):**

| Angle | Metrics | Logs | Traces | Dashboard |
|-------|---------|------|--------|-----------|
| Serverless | CloudWatch + Embedded Metrics | CloudWatch Logs | X-Ray | CW Dashboard |
| K8s | Prometheus + KEDA | Loki / CW | Jaeger | Grafana |
| Managed | CloudWatch native | CloudWatch Logs | CloudWatch ServiceLens | CW Dashboard |

**Metrics CDO phải có (TF3-specific):**
- `incidents_detected_total` per pattern per tenant
- `incidents_auto_resolved_total` vs `incidents_escalated_total`
- `ai_detect_latency_p99`, `ai_decide_latency_p99`, `ai_verify_latency_p99`
- `idempotency_lock_hits_total` (track double-execute prevention)
- `circuit_breaker_open_total` per pattern
- `dry_run_executions_total`
- `k8s_action_duration_ms` (từ lúc CDO gọi K8s API đến pod healthy)

### Chiều: AI Integration

**AI skeleton đã live từ T5 W11. T2 W12: tích hợp đầy đủ vào CDO flow:**

```
Client Request
    ↓
CDO Platform (auth + rate limit + tenant routing)
    ↓
AI Engine Endpoint (POST /v1/<path>)
    ↓
CDO Response processing (parse + audit log + notify)
    ↓
Client Response
```

**Integration checklist TF3:**
- [ ] Alert ingestor → `/v1/detect` call với đúng payload (correlation_id, idempotency_key, tenant_id)
- [ ] `dry_run_mode` flag propagated correctly đến AI engine
- [ ] `/v1/decide` response: CDO parse `action_plan[]` + check `blast_radius_ok`
- [ ] CDO execute K8s action (via ServiceAccount + RBAC): `kubectl patch` / `kubectl rollout restart` / `kubectl scale`
- [ ] DynamoDB idempotency lock: conditional write trước execute, release sau verify
- [ ] `/v1/verify` call: CDO trigger sau X giây, handle `regression` → rollback
- [ ] Audit log: every AI call ghi vào S3 Object Lock với đủ fields
- [ ] Escalation: nếu verify = fail → AI-generated message gửi Slack webhook với context bundle
- [ ] Circuit breaker: sau N consecutive failures per pattern → open circuit

**16h - Curveball #2 medium (30p) - TF3 examples:**
> - "Queue backlog pattern cần scale up worker đến 3x, không phải 2x" → CDO: blast_radius config update + ADR
> - "DynamoDB conditional write chuyển sang Redis SETNX" → CDO: Terraform module update, Deployment contract impact
> - "Concurrent incident handling: engine phải handle 5 incidents simultaneously" → CDO: concurrency config + test

Response format:
1. Đọc curveball → identify impact (5 phút)
2. Quyết định: (a) infra change gì, (b) contract sửa không (nếu schema change)
3. Viết response vào `curveball-responses.md` ngay
4. Update ADR nếu cần
5. Tạo Jira task để execute change

**Jira tasks T2:**
- Task: "CI/CD pipeline setup" → evidence: pipeline run URL
- Task: "Observability stack live" → evidence: dashboard screenshot
- Task: "AI endpoint integration code" → evidence: commit SHA
- Task: "Curveball #2 response" → evidence: curveball-responses.md commit

---

## T3 30/06 - E2E Test + Integration Session

### Mục tiêu ngày

1. E2E test per platform (CDO-specific scenarios)
2. **14h-16h Integration session: call AI endpoint THẬT**
3. Tenant onboarding flow working

### Sáng: E2E Test + Scenario Simulation Setup

**TF3 yêu cầu ≥10 scenarios trong ≥4h window - đây là hard requirement.**

Build **incident injection script** cho sandbox cluster:

```bash
# inject_incident.sh - gợi ý
# OOM kill: patch deployment với memory limit thấp
kubectl patch deployment $SVC -n $NS -p '{"spec":{"containers":[{"name":"app","resources":{"limits":{"memory":"10Mi"}}}]}}'

# CrashLoopBackOff: inject bad command
kubectl patch deployment $SVC -n $NS -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","command":["/bin/sh","-c","exit 1"]}]}}}}'

# Queue backlog: manual metric injection hoặc scale consumer xuống 0
kubectl scale deployment queue-consumer -n $NS --replicas=0
```

**12 scenarios CDO phải chạy (xem chi tiết ở `01_task_force_analysis.md`):**

| # | Inject | Expected CDO action | Pass/Fail criteria |
|---|--------|--------------------|--------------------|
| S1 | OOM pod | Memory limit patched, pod restart | Pod Running trong 2 phút |
| S2 | OOM persistent ×3 | Circuit breaker → escalate | Slack message received |
| S3 | CrashLoopBackOff | Rollout restart | Pod Running trong 2 phút |
| S4 | CrashLoop persistent | Circuit breaker open | No more K8s action |
| S5 | Queue backlog | Worker scaled up | Backlog metric drops |
| S6 | Queue + blast radius exceeded | Escalate direct | No K8s action taken |
| S7 | dry_run = true | Log only | No K8s action, audit logged |
| S8 | Unknown pattern low confidence | Escalate | Message sent to Slack |
| S9 | Verify: regression | Auto rollback | Previous state restored |
| S10 | Tenant A incident | OOM fix tenant A only | Tenant B namespace clean |
| S11 | Concurrent: OOM + CrashLoop | Idempotency lock | No double-execute |
| S12 | Engine 503 | CDO fallback + circuit breaker | No crash, audit logged |

**Auto-resolve rate calculation:**
```
auto_resolved = scenarios where AI action succeeded without escalation
auto_resolve_rate = auto_resolved / total_scenarios ≥ 60%
```

**Scenario simulation window ≥4h:**
- T3 sáng: inject S1-S6 (2h window)
- T3 chiều sau integration: inject S7-S12 (2h window)
- Capture timeline + outcomes vào `test_simulation_log.md`

**Tenant onboarding test TF3:**
- [ ] `kubectl create namespace tenant-c` + apply RBAC → < 30 phút end-to-end
- [ ] Verify: tenant-c không thấy được events từ tenant-a, tenant-b

### 14h-16h: Integration Session (CDO call AI endpoint THẬT)

**Đây là dependency thật. Chuẩn bị kỹ:**

Pre-session checklist:
- [ ] AI engine URL confirmed (không phải skeleton nữa, thật)
- [ ] Auth token/config ready
- [ ] Test payload theo đúng AI API contract schema
- [ ] Logging configured để capture round-trip

**Session protocol TF3:**
```
1. CDO gọi /v1/detect với real OOM alert payload (10 phút)
2. Verify response: anomaly_type đúng không? confidence > threshold?
3. CDO gọi /v1/decide với detect output (10 phút)
4. Verify: action_plan có lệnh K8s thực tế không? blast_radius_ok return đúng?
5. CDO execute K8s action (dry_run=true trước, rồi false)
6. CDO gọi /v1/verify sau 30s (10 phút)
7. Verify: status = success? Audit log ghi vào S3?
8. Test multi-tenant: Tenant A incident không leak sang Tenant B
9. Test idempotency: gửi cùng idempotency_key 2 lần → DynamoDB lock chặn không?
10. Document: pass/fail per step + latency từng endpoint
```

**Nếu AI engine chưa work hoàn toàn:**
- Ghi rõ: "AI partial response, thiếu field X" → document gap
- Tiếp tục build CDO infra với mock cho remaining E2E
- Escalate mentor nếu AI hoàn toàn không work

**16h Task Force Sync:**
- AI nhóm + CDO leads cùng review integration results
- Identify gaps cần fix trước T4

**Jira tasks T3:**
- Task: "E2E test scenarios" → evidence: test run output / screenshot
- Task: "Tenant onboarding flow test" → evidence: timing evidence
- Task: "Integration session results" → evidence: test log / doc

---

## T4 01/07 - Polish + Chaos

### Mục tiêu ngày

1. Polish E2E scenarios
2. Cập nhật docs: cost_analysis (measured), infra_design (updated), ADRs
3. **14h Curveball #3 chaos (60p)**
4. 17-22h dry-run + final bug fix
5. **18h Pack #2 submit + code freeze**

### Sáng: Polish + Doc

**Infra polish checklist TF3:**
- [ ] 5 safety checkpoints đều working và tested (dry-run, blast-radius, verify, rollback, circuit breaker)
- [ ] Audit log queryable qua Athena: `SELECT * FROM audit_log WHERE tenant_id = 'tenant-a' LIMIT 10`
- [ ] Auto-resolve rate ≥ 60%: tính từ scenario simulation results
- [ ] Multi-tenant RBAC isolation test pass (Tenant A không access Tenant B namespace)
- [ ] Idempotency: double-submit same `idempotency_key` → DynamoDB lock prevents second execute
- [ ] Escalation demo: AI-generated message với context bundle gửi Slack webhook
- [ ] Security scan: 0 CRITICAL CVE

**Doc updates sáng T4 (TF3-specific):**

1. **`05_cost_analysis.md` - measured actual**
   - [ ] EKS cluster cost (control plane + nodes) per 2 tuần
   - [ ] DynamoDB read/write units actual
   - [ ] S3 Object Lock storage cost
   - [ ] AI inference cost (3 endpoint calls × scenario count)
   - [ ] Compare với CDO kia: "K8s operator always-on vs Step Functions pay-per-exec"

2. **`07_test_eval_report.md` - NEW (TF3)**
   - [ ] §2: SLO evidence (p99 latency per endpoint, error rate)
   - [ ] §3: Scenario simulation results: 12 scenarios, auto-resolve rate
   - [ ] §4: Multi-tenant isolation test (all 4 cases pass)
   - [ ] §5: Security scan results
   - [ ] §6: Failure analysis (especially circuit breaker behavior)
   - [ ] Chaos curveball responses

3. **`08_adrs.md` TF3 target ≥5 ADRs:**
   - ADR-001: Infra angle (K8s Operator / Step Functions / Serverless)
   - ADR-002: Compute target (AI engine hosting)
   - ADR-003: Audit storage (S3 Object Lock vs append-only DB)
   - ADR-004: Idempotency lock (DynamoDB vs Redis)
   - ADR-005: CI/CD strategy (per brief yêu cầu 5 ADR cho TF3)
   - ADR-006+: Từ W12 iteration (observability, multi-tenant pattern...)

### 14h - Curveball #3 Chaos (60p) - TF3 Examples

**Possible chaos scenarios TF3:**
- **"EKS control plane unreachable 30 phút"** → CDO: circuit breaker open, queue incidents, retry khi recover
- **"AI engine OOM khi xử lý 5 incidents concurrent"** → CDO: scale AI compute, adjust concurrency limit, Deployment contract impact?
- **"S3 Object Lock bucket region fail"** → CDO: fallback audit storage, dual-write strategy
- **"DynamoDB conditional write throttle"** → CDO: retry with backoff, fallback idempotency check

**Response process (60 phút):**
```
0-10p: Đọc + assess: cái gì break? Safety checkpoint nào affected?
10-25p: Decide response: circuit breaker / scale / fallback / rollback
25-45p: Execute: Terraform change hoặc code change
45-55p: Verify: hệ thống phục hồi không? Audit log vẫn chạy không?
55-60p: Write curveball-responses.md entry + update ADR nếu architecture đổi
```

**curveball-responses.md entry format:**
```markdown
## Curveball #3 - <title>
- **Time**: 14:00 - 15:00 T4 01/07
- **Description**: <what was injected>
- **Impact assessment**: <what breaks, what doesn't>
- **Response**: <what we did>
- **ADR update**: <if architecture decision changed>
- **Contract impact**: <if any contract change needed>
- **Outcome**: Pass / Partial / Fail
- **Lessons**: <what we'd do differently>
```

### 16h-17h: Final Polish

- [ ] Fix bugs phát hiện từ curveball
- [ ] Update infra_design.md nếu architecture thay đổi
- [ ] Slides draft review (CDO section: 15 phút pitch)
- [ ] individual-pitches.md: mỗi người viết 2-3 câu về contribution của mình

### 17h-22h: Dry-Run + Buffer

**Dry-run internal (17h-19h):**
- CDO lead trình bày như thật (15 phút)
- Team hỏi mock Q&A (10 phút)
- Feedback + fix (15 phút)

**Individual defense prep (19h-21h):**
- Mỗi người review lại Jira tasks mình Done
- Chuẩn bị walk-through cho 2-3 task phức tạp nhất
- Format: "Task X: tôi làm gì → commit SHA → decision tôi đưa ra → trade-off"

**Buffer fix bugs (21h-22h):**
- Chỉ fix bugs P0 (block demo)
- Không thêm feature mới

### 18h - Pack #2 Submit + Code Freeze

**Final checklist trước 18h T4:**

Documentation:
- [ ] `docs/01_requirements_analysis.md` refined
- [ ] `docs/02_infra_design.md` updated với W12 changes
- [ ] `docs/03_security_design.md` refined
- [ ] `docs/04_deployment_design.md` working (CI/CD live evidence)
- [ ] `docs/05_cost_analysis.md` measured actual ✅
- [ ] `docs/07_test_eval_report.md` with SLO evidence ✅
- [ ] `docs/08_adrs.md` ≥5 ADRs final
- [ ] `curveball-responses.md` - 3 curveballs documented
- [ ] `individual-pitches.md` - mỗi member điền

Build artifacts:
- [ ] `final-build/` - IaC + manifests complete
- [ ] Platform infra deployed + integrated với AI engine
- [ ] E2E demo chạy được
- [ ] `SLIDES.pdf` (hoặc SLIDES.pptx + export PDF)
- [ ] `demo-video.mp4` recorded

Process:
- [ ] `retrospective.md` - quick retrospective của team
- [ ] Jira: tất cả task Done có evidence link
- [ ] **git tag `final`**
- [ ] **CODE FREEZE 18h T4** - sau đây chỉ sửa slides + script

---

## T5 02/07 - Pitch Day

### 8h: Hard Code Freeze

Sau 8h T5: không push code. Chỉ sửa slides + script + rehearse.

### Chuẩn bị trước giờ pitch

- [ ] Demo environment warm-up (không cold start lúc demo)
- [ ] Test demo flow 1 lần nữa
- [ ] Backup plan nếu live demo fail (recording sẵn)
- [ ] Mỗi người review 2-3 task sẽ defend

### Format pitch CDO TF3 (15 phút/nhóm)

```
[5 phút] Architecture overview
  - Angle: "Tôi chọn [K8s Operator / Step Functions] vì TF3 về bản chất là K8s remediation..."
  - Architecture diagram: alert → detect → decide → execute → verify → audit
  - 5 safety checkpoints: dry-run, blast-radius, verify, rollback, circuit breaker
  - Differentiation: "CDO kia dùng X, tôi dùng Y - vượt trội ở axis Z"

[5 phút] Demo live
  - Inject OOM incident → CDO detect → decide → execute K8s action → verify → audit logged
  - Hiện Athena query: "SELECT * FROM audit_log..." → kết quả
  - Hiện escalation message với context bundle nếu scenario fail

[5 phút] Q&A panel
  - "Sao chọn K8s Operator / Step Functions?" → cite ADR-001
  - "5 safety checkpoints implement thế nào?" → walk through code
  - "Auto-resolve rate?" → cite test_simulation_log.md (≥60%)
  - "Audit tamper-evident verify thế nào?" → S3 Object Lock config + Athena demo
  - "Curveball #3?" → cite curveball-responses.md
```

### Điểm differentiation TF3 phải nói rõ

Mentor sẽ hỏi: "CDO [M] vs CDO [M'] trong TF3, vượt trội ở đâu?"

**3 axis TF3-specific:**
1. **Safety depth**: "5 safety checkpoints tôi implement đầy đủ. Chứng minh: dry-run test + circuit breaker open scenario"
2. **Audit quality**: "S3 Object Lock + Athena queryable. Query demo ngay: response time X giây, retention configured"
3. **Ops angle**: "[K8s Operator = GitOps-native, drift detection] / [Step Functions = visual audit trail, debug-friendly] so với CDO kia [angle B] → tôi win ở axis [reliability/ops/cost]"

---

## Risk mitigation W12 TF3-specific

| Risk | Dấu hiệu | Mitigation |
|------|----------|------------|
| EKS setup broken sau curveball | `kubectl get nodes` fail | Rollback Terraform, không cố fix live |
| AI engine fail T3 integration | 3 endpoints không trả response đúng schema | Continue với mock, document gap, escalate |
| Auto-resolve rate < 60% | S6-S8 scenarios không pass | Identify pattern fail root cause, fix logic CDO decision gate |
| Audit log không queryable | Athena query error | Glue crawler re-run, Athena workgroup check |
| Idempotency lock not working | Double-execute same incident | DynamoDB conditional write test, fix before demo |
| Member ôm hết task | 1 người > 60% K8s commits | Assign: 1 người = 1 pattern implementation |
| Demo fail (EKS pod crash) | Pod not Ready | Warm up demo cluster T5 sáng trước pitch |
