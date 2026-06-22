# Jira Task Breakdown - CDO Team · TF3 Self-Heal Engine
> Task list theo ngày, size chuẩn (30 phút - 1 ngày), sẵn sàng pull vào Jira backlog.
> Mỗi task Done PHẢI có evidence link (commit SHA / PR URL / screenshot).

---

## Nguyên tắc Jira (từ brief mentor)

- Task size: **30 phút → 1 ngày**. Nhỏ hơn → gộp. Lớn hơn → split.
- Flow: To Do → **In Progress** → **In Review** → Done
- **Daily comment** ít nhất 1 lần/ngày: progress / blocker / ETA
- **Evidence link khi close**: commit SHA, PR URL, doc link, screenshot
- Anti-gaming: không backdating, không batch-Done, không Done ngay sau Assigned

---

## W11 - Task List

### Epic: BA + Discovery (T2-T3 W11)

| ID | Task | Size | Owner | Evidence |
|----|------|------|-------|---------|
| W11-01 | Đọc `TF3_SELFHEAL_LEARNER.md` + note 15 câu hỏi | 1h | All | Doc: question list commit |
| W11-02 | Team sync merge câu hỏi, assign mảng cho interview | 30p | Lead | Doc: merged list commit |
| W11-03 | Client interview T2 chiều + ghi notes | 2h | All | Doc: interview notes commit |
| W11-04 | Viết debrief "Tôi hiểu là..." + confirm 3 patterns + blast radius | 1h | Lead | Debrief commit + mentor confirm screenshot |
| W11-05 | Draft `01_requirements_analysis.md` §1 (TF3 context) + §2 (NFR) | 2h | Lead | Commit SHA |
| W11-06 | Draft `01_requirements_analysis.md` §3 (angle) + §4 (comparison) | 1h | Lead | Commit SHA |
| W11-07 | Announce differentiation angle cho AI nhóm (angle + compute + ServiceAccount name) | 30p | Lead | Slack/email screenshot |

### Epic: Architecture Design (T3-T4 W11)

| ID | Task | Size | Owner | Evidence |
|----|------|------|-------|---------|
| W11-08 | Draft Mermaid flow diagram: alert → detect → decide → execute → verify → audit | 1.5h | Infra Lead | Commit SHA |
| W11-09 | Draft `02_infra_design.md` §1-§3 (arch diagram, component table, angle deep-dive) | 3h | Infra Lead | Commit SHA |
| W11-10 | Draft `02_infra_design.md` §4 (multi-tenant: namespace-per-tenant) + §5-§7 | 2h | Infra Lead | Commit SHA |
| W11-11 | Write ADR-001: Infra angle (K8s Operator vs Step Functions vs Serverless) | 45p | Lead | Commit SHA |
| W11-12 | Write ADR-002: Compute target cho AI engine (Fargate vs EKS pod) | 45p | Infra | Commit SHA |
| W11-13 | Write ADR-003: Audit storage (S3 Object Lock vs append-only DB) | 45p | Infra | Commit SHA |
| W11-14 | Write ADR-004: Idempotency lock (DynamoDB vs Redis) | 30p | Backend | Commit SHA |
| W11-15 | Draft `03_security_design.md` §1 (network, SG) + §2 (IAM + K8s RBAC) + §3 (secrets) | 2h | Security | Commit SHA |
| W11-16 | Draft `04_deployment_design.md` §1 (IaC structure) + §2 (CI/CD pipeline) | 1.5h | DevOps | Commit SHA |

### Epic: Contract Review + Co-Design (T4-T5 W11)

| ID | Task | Size | Owner | Evidence |
|----|------|------|-------|---------|
| W11-17 | Review Telemetry Contract: alert webhook format, K8s events format, volume | 1h | Lead + Ops | Push-back notes commit |
| W11-18 | Review AI API Contract: 3 endpoints schema, `dry_run_mode`, `blast_radius_ok`, latency SLA | 1h | Lead + Backend | Push-back notes commit |
| W11-19 | Review Deployment Contract: K8s ServiceAccount, kubeconfig secret, DynamoDB lock, S3 bucket | 1h | Lead + Infra | Push-back notes commit |
| W11-20 | Co-design session T5 sáng với AI nhóm (negotiate 3 contracts) | 3h | Lead + AI Liaison | Final contract commit SHA |
| W11-21 | Onsite T5: present design doc + curveball #1 response | 2h | All | curveball-responses.md commit |
| W11-22 | Update ADR nếu curveball #1 thay đổi architecture decision | 30p | Lead | Commit SHA |

### Epic: Base Infra Build (T5 chiều - T6 W11)

| ID | Task | Size | Owner | Evidence |
|----|------|------|-------|---------|
| W11-23 | Terraform init: S3 remote state + DynamoDB lock + module structure | 1h | Infra Lead | Commit SHA |
| W11-24 | Terraform module `networking`: VPC, private/public subnets, SG, VPC endpoints | 2h | Infra | Commit SHA + `tf plan` output |
| W11-25 | Terraform module `eks-cluster`: EKS cluster, managed node group, OIDC provider | 2h | Infra Lead | Commit SHA + `kubectl get nodes` |
| W11-26 | Terraform module `eks-rbac`: namespaces (tenant-a, tenant-b), ServiceAccount, ClusterRole, RoleBinding | 1.5h | Security | Commit SHA + `kubectl describe sa` |
| W11-27 | Terraform module `audit-storage`: S3 Object Lock bucket, Glue DB + crawler, Athena workgroup | 1.5h | Backend | Commit SHA + S3 console screenshot |
| W11-28 | Terraform module `idempotency-lock`: DynamoDB table (idempotency_key PK) | 30p | Backend | Commit SHA + DynamoDB console screenshot |
| W11-29 | Terraform module `ai-engine`: Fargate task / EKS pod cho AI skeleton | 2h | Infra | Commit SHA + running proof |
| W11-30 | Alert ingestor: API GW + Lambda nhận AlertManager webhook → log | 1h | Backend | CloudWatch log showing webhook received |
| W11-31 | Smoke test: alert webhook → ingest → `/v1/detect` skeleton call → 200 + dummy JSON | 30p | Backend | Screenshot response JSON (3 fields: anomaly_type, confidence, correlation_id) |
| W11-32 | RBAC test: `kubectl auth can-i` - verify SA không có cluster-admin | 30p | Security | `kubectl auth can-i` output screenshot |
| W11-33 | Draft `05_cost_analysis.md` skeleton (EKS + DynamoDB + S3 + AI forecast) | 1h | Ops | Commit SHA |
| W11-34 | Standup notes W11 maintain + daily commit | Daily 10p | All | Commit SHA daily |
| W11-35 | Pack #1 final review: checklist + commit + verify git history spread | 1h | Lead | Commit SHA + git log screenshot |

---

## W12 - Task List

### Epic: CI/CD + Safety Checkpoints (T2 W12)

| ID | Task | Size | Owner | Evidence |
|----|------|------|-------|---------|
| W12-01 | GitHub Actions: build + test pipeline (Terraform validate + unit test) | 2h | DevOps | Pipeline run URL |
| W12-02 | GitHub Actions: Terraform plan on PR + OIDC auth (no static keys) | 1h | DevOps | PR screenshot |
| W12-03 | GitHub Actions: Gitleaks scan + Trivy image scan | 1h | Security | Pipeline run URL |
| W12-04 | Safety checkpoint 1: Dry-run mode - CDO check `dry_run_mode` flag, skip K8s action | 1.5h | Backend | Commit SHA + test log showing no K8s action |
| W12-05 | Safety checkpoint 2: Blast-radius gate - CDO parse `blast_radius_ok`, route to escalate | 1h | Backend | Commit SHA + test log |
| W12-06 | Safety checkpoint 3: Verify post-act - CDO call `/v1/verify` after execute | 1h | Backend | Commit SHA |
| W12-07 | Safety checkpoint 4: Auto rollback - CDO apply pre-state on regression | 2h | Backend | Commit SHA + before/after K8s state |
| W12-08 | Safety checkpoint 5: Circuit breaker - track consecutive failures, open after N | 1.5h | Backend | Commit SHA + test showing CB open |
| W12-09 | Observability: metrics dashboard (incidents_resolved, circuit_breaker, latency per endpoint) | 1.5h | Ops | Dashboard screenshot |
| W12-10 | Observability: CloudWatch/Grafana alarms (circuit breaker open → alert) | 1h | Ops | Alarm config commit |
| W12-11 | AI 3-endpoint integration: full flow code (detect → gate → decide → execute → verify) | 3h | Backend | Commit SHA |
| W12-12 | DynamoDB idempotency lock integration: conditional write before execute, release after verify | 1.5h | Backend | Commit SHA + test log |
| W12-13 | Audit log: every AI call → S3 Object Lock write (fields: timestamp, tenant_id, correlation_id, action, outcome) | 1.5h | Backend | Commit SHA + audit log sample in S3 |
| W12-14 | Escalation: AI-generated message + context bundle → Slack webhook | 1h | Backend | Slack screenshot |
| W12-15 | Respond curveball #2 + update ADR + implement if needed | 30p | Lead | curveball-responses.md commit |

### Epic: Scenario Simulation + Integration (T3 W12)

| ID | Task | Size | Owner | Evidence |
|----|------|------|-------|---------|
| W12-16 | Build incident injection script: OOM / CrashLoop / Queue backlog inject | 2h | Backend | Commit SHA + script README |
| W12-17 | Run S1-S6: OOM normal/persistent, CrashLoop normal/persistent, Queue normal/blast-radius | 2h | Backend + Ops | test_simulation_log.md commit |
| W12-18 | Run S7-S12: dry-run, low confidence, regression/rollback, multi-tenant, concurrent, engine-down | 2h | Backend + Ops | test_simulation_log.md update commit |
| W12-19 | Calculate auto-resolve rate: S_resolved / 12 ≥ 60% | 30p | Lead | Commit SHA + rate calculation |
| W12-20 | Integration session T3 14h: call AI real 3 endpoints + document results per endpoint | 2h | Lead + Backend | Integration test log commit |
| W12-21 | Tenant onboarding: new namespace + RBAC < 30 phút, isolation verified | 1h | DevOps | Timing evidence + `kubectl auth can-i` output |
| W12-22 | Athena audit log query test: `SELECT * FROM audit_log WHERE tenant_id = 'tenant-a'` | 30p | Backend | Athena query result screenshot |
| W12-23 | Write ADR-005: CI/CD strategy | 30p | DevOps | Commit SHA |
| W12-24 | Write ADR-006: Observability stack | 30p | Ops | Commit SHA |

### Epic: Polish + Docs (T4 W12)

| ID | Task | Size | Owner | Evidence |
|----|------|------|-------|---------|
| W12-25 | Security scan: Trivy + fix CRITICAL CVE + RBAC final verify | 1h | Security | Scan report + commit SHA |
| W12-26 | Load/stress test: 10 concurrent incidents, verify idempotency holds | 1h | Ops | Test output log |
| W12-27 | Multi-tenant isolation final test: all 4 cases pass (cross-namespace data leak attempt) | 1h | Security | Test output showing blocks |
| W12-28 | Update `02_infra_design.md` with W12 changes (safety checkpoints, circuit breaker added) | 1h | Infra | Commit SHA |
| W12-29 | Write `05_cost_analysis.md` measured actual: EKS nodes, DynamoDB, S3, AI calls | 1.5h | Ops | Commit SHA + AWS Cost Explorer screenshot |
| W12-30 | Write `07_test_eval_report.md`: SLO, 12 scenario results, auto-resolve rate, chaos response | 2h | Lead | Commit SHA |
| W12-31 | Update ADR-007, ADR-008 from W12 decisions (circuit breaker threshold, scenario simulation method) | 1h | Lead | Commit SHA |
| W12-32 | Respond curveball #3 + infra fix + verify system recovers | 2h | Infra + Backend | curveball-responses.md commit |
| W12-33 | Update docs post-curveball #3 | 1h | Lead | Commit SHA |
| W12-34 | Write `individual-pitches.md` (mỗi người 2-3 câu về contribution) | 30p | All | Commit SHA |
| W12-35 | Write `retrospective.md` | 30p | Lead | Commit SHA |
| W12-36 | Slides: CDO section (5p arch + 5p demo + differentiation story) | 2h | Lead + All | SLIDES.pdf commit |
| W12-37 | Record demo video: inject OOM → CDO detect/decide/execute/verify → audit queryable | 1h | Lead | demo-video.mp4 commit |
| W12-38 | Dry-run internal: 15p pitch + 10p mock Q&A (5 safety checkpoints, RBAC, audit) | 2h | All | standup-notes.md update |
| W12-39 | Individual defense prep: mỗi người review 3 task, chuẩn bị walk-through | 1h | All | standup-notes.md update |
| W12-40 | git tag `final` + code freeze confirm + git log check | 15p | Lead | `git show final` screenshot |

---

## Phân công gợi ý (team 3-4 người)

| Role | Focus area | Key W11 tasks | Key W12 tasks |
|------|-----------|---------------|---------------|
| **CDO Lead** | Architecture, ADRs, contracts, slides | 01,04,07,11,20,21 | 15,20,30,31,36,37 |
| **Infra Engineer** | EKS, Terraform, networking | 08,09,10,23,24,25,29 | 28,32 |
| **Backend Engineer** | Integration code, safety checkpoints, audit | 26,27,28,30,31 | 04-13,16-18,22 |
| **DevOps/Security** | CI/CD, RBAC, observability, security scan | 15,16,26,32 | 01-03,09,10,21,25,27 |

> Nếu team 3 người: merge DevOps/Security vào Backend hoặc Infra.

---

## Commit message convention TF3

```
feat(eks): provision cluster + managed node group + OIDC
feat(rbac): namespace tenant-a tenant-b + ClusterRole least-priv
feat(audit): S3 Object Lock bucket + Glue crawler + Athena workgroup
feat(safety): circuit breaker - open after 3 consecutive failures
feat(integration): detect-decide-execute-verify flow code
test(simulation): inject OOM scenario S1 - auto-resolved in 90s
docs(02_infra): add 5 safety checkpoints section
adr(005): CI/CD strategy - GitHub Actions + OIDC
fix(idempotency): DynamoDB conditional write key format mismatch
```

## Evidence checklist TF3-specific

- EKS cluster: `kubectl get nodes -o wide` output
- RBAC: `kubectl auth can-i patch pods -n tenant-a --as=system:serviceaccount:tenant-a:self-heal-engine-sa`
- RBAC isolation: `kubectl auth can-i get pods -n tenant-b --as=...tenant-a...` → no
- S3 Object Lock: AWS console showing Object Lock enabled + mode
- DynamoDB: table exists + sample conditional write test log
- Scenario simulation: `test_simulation_log.md` với timestamp, scenario, outcome, auto-resolve rate
- Athena query: screenshot query result with timestamp, tenant_id, action fields
- Escalation: Slack screenshot showing AI-generated context bundle message
- Circuit breaker: log showing CB open + subsequent incidents routed to escalate
