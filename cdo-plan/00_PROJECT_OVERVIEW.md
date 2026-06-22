# CDO Team - Project Overview & Analysis
> Capstone Phase 2 · W11-W12 · **Task Force 3 - Self-Heal Engine**
> Tài liệu này là working doc của team CDO, tổng hợp từ brief mentor và phân tích nội bộ.
> Cập nhật liên tục theo build progress.

---

## 1. Bức tranh tổng thể

### Chúng ta là ai trong bức tranh này?

```
Task Force (TF)
├── 1 Nhóm AI  → own đề tài, design AI engine, publish 3 contracts
└── 2-3 Nhóm CDO (chúng ta) → build infra platform hosting AI engine
                                compete nhau trên execution quality
```

Team CDO **không** quyết đề tài (do AI quyết), nhưng **hoàn toàn tự chủ** về:
- Architecture angle (serverless / K8s / streaming / lakehouse / managed observability)
- IaC + CI/CD strategy
- Security + observability approach
- Cost optimization
- Multi-tenant model

**Win condition**: CDO nào ra được story "infra của tôi vượt trội hơn CDO kia ở axis X, với evidence cụ thể" → Tier 1-2.

---

## 2. Đề tài của chúng ta: TF3 - Self-Heal Engine

**Client**: VP Engineering · SaaS platform B2B · 200+ microservice · EKS multi-AZ · us-east-1

**Vấn đề core**: Mỗi đêm on-call nhận 2-4 page, **80% là known patterns** lặp đi lặp lại (Pod OOMKilled, service stuck, queue backlog, cert expiring). Engineer thức 2h sáng chỉ để click "restart". eNPS rớt 42→11, retention -30% YoY.

**Solution cần build**:
```
detect → match runbook → execute (audited) → verify → escalate nếu fail
```

**Hard requirements CDO phải support**:
| Requirement | Infra implication |
|-------------|------------------|
| ≥3 patterns implemented + tested | Sandbox EKS với workload inject + K8s API access |
| Auto-resolve rate ≥60% trên ≥10 scenarios | Test harness + scenario simulation ≥4h window |
| Zero unsafe action | RBAC least-privilege + blast-radius enforcement |
| Audit log tamper-evident, retention ≥90 days | S3 Object Lock + Athena query |
| 5 safety checkpoints | dry-run · blast-radius · verify · auto rollback · circuit breaker |
| Multi-tenant ≥2 tenants RBAC isolation | Namespace-per-tenant + K8s RBAC |
| AI engine: 3 endpoints `/detect` `/decide` `/verify` | CDO route + integrate 3 endpoints |

**TF3 có 2 CDO compete** - differentiation angle là critical.

Xem phân tích chi tiết TF3: [`01_task_force_analysis.md`](01_task_force_analysis.md)

---

## 3. Timeline CDO cần deliver

### W11 (22/06 - 26/06) - Tuần tài liệu + base infra

| Ngày | CDO cần làm |
|------|-------------|
| T2 22/06 | Đọc đề TFx, chuẩn bị 10-15 câu hỏi, interview Client chiều |
| T3 23/06 | Viết `01_requirements_analysis.md`, xác định **differentiation angle**, thông báo angle cho AI nhóm |
| T4 24/06 | `02_infra_design.md` draft + `08_adrs.md` ≥2 ADR. Review contract draft từ AI. Push-back nếu cần. |
| T5 25/06 | Sáng: co-design contracts với AI. Chiều onsite: approve tài liệu + **ký 3 contracts** |
| T6 26/06 | **EOD Pack #1 deadline**: base infra chạy được (VPC + cluster + observability) |

**Deliverable CDO W11 EOD T6:**
- [ ] `docs/01_requirements_analysis.md` ✅
- [ ] `docs/02_infra_design.md` ✅
- [ ] `docs/03_security_design.md` (draft)
- [ ] `docs/04_deployment_design.md` (draft)
- [ ] `docs/05_cost_analysis.md` (skeleton)
- [ ] `docs/08_adrs.md` ≥3 ADRs
- [ ] `infra/` - Terraform base VPC + cluster + observability chạy được
- [ ] `standup-notes.md`

### W12 (29/06 - 03/07) - Tuần build + integrate

| Ngày | CDO cần làm |
|------|-------------|
| T2 29/06 | CI/CD canary, observability stack, integrate AI endpoint (skeleton) |
| T3 30/06 | **Integration session**: call AI endpoint THẬT (không mock nữa), E2E test per platform |
| T4 01/07 | Polish E2E. **16h Curveball #3 chaos (60p)**. Respond + update docs. Code review + fix. |
| T5 02/07 | **8h CODE FREEZE**. Git tag `final`. Onsite pitch |

**Deliverable CDO W12 EOD T4 (code freeze 18h):**
- [ ] `final-build/` IaC + manifests integrated
- [ ] `docs/05_cost_analysis.md` measured actual
- [ ] `docs/07_test_eval_report.md` SLO evidence + chaos response
- [ ] `SLIDES.pdf`
- [ ] `demo-video.mp4`
- [ ] `curveball-responses.md`
- [ ] `individual-pitches.md`
- [ ] `retrospective.md`

---

## 4. 3 Contracts TF3 - CDO phải nắm rõ

AI draft, CDO review và push-back. Sau khi ký T5 W11 → **FREEZE**.

| Contract | Điểm đặc thù TF3 | CDO phải push-back nếu |
|----------|------------------|------------------------|
| **Telemetry Contract** | CDO emit K8s events (pod state, metric changes) + alert webhook payload | Volume quá cao, format khó emit từ Prometheus AlertManager |
| **AI API Contract** | 3 endpoints: `/v1/detect`, `/v1/decide`, `/v1/verify`. Mandatory fields: `idempotency_key`, `dry_run_mode`, `correlation_id` | P99 latency SLA không feasible, `dry_run_mode` format (header vs body) |
| **Deployment Contract** | Yêu cầu đặc thù TF3: `kubeconfig` secret, K8s ServiceAccount + RBAC least-privilege, idempotency lock (DynamoDB), S3 Object Lock cho audit | Compute spec quá lớn, secrets path không match CDO naming, network access K8s API server |

**AI API 3-endpoint flow CDO phải integrate:**
```
Alert webhook fires
    ↓ CDO ingest
POST /v1/detect  →  { anomaly_type, confidence, correlation_id }
    ↓ CDO decision gate
POST /v1/decide  →  { action_plan[], blast_radius_ok, dry_run_result }
    ↓ CDO execute (nếu blast_radius_ok + không dry_run)
    ↓ K8s API action
POST /v1/verify  →  { status: success|regression|next_action }
    ↓ CDO: audit log + escalate nếu fail
```

**Từ T6 W11**: Gọi skeleton endpoint trả dummy JSON đúng schema 3 endpoints.
**T3 W12**: Gọi AI engine thật - đây là dependency thật duy nhất.

---

## 5. Scoring CDO - Hiểu để build đúng

| Pillar | Weight | Cần làm gì |
|--------|--------|------------|
| W11 Platform Design Doc | 25% tổng | Doc chất lượng cao, angle rõ ràng |
| W11 Contract Acceptance | 20% tổng | Co-design tốt, push-back có lý |
| W11 Base Infra Ready | 35% tổng | Chạy được EOD T6 |
| W11 Task force Sync | 10% tổng | Angle khác CDO còn lại trong TF |
| W12 Infrastructure Quality | 40% tổng | IaC + GitOps + observability + security |
| W12 AI Engine Integration | 25% tổng | E2E demo chạy thật |
| W12 Present Performance | 25% tổng | Differentiation story thuyết phục |
| W12 Individual Defense | 15% tổng | Walk through commit + decision |

**Anti-theater rules cần nhớ:**
- Pitch xuất sắc nhưng ≥50% HV defense fail → cap **Tier 3**
- CDO differentiation thuyết phục yếu → cap **Tier 2**
- Free-rider (< 5 task Done, không evidence link) → individual cap **Tier 3**

---

## 6. Curveball CDO TF3 phải sẵn sàng

TF3 đặc thù: curveball thường liên quan K8s cluster, audit, hoặc pattern scope.

| # | Thời điểm | Ví dụ TF3-specific | CDO response |
|---|-----------|-------------------|--------------|
| #1 Nhẹ | T5 W11 chiều (15p) | "Thêm pattern: Cert expiring → rotate secret" hoặc "COMPLIANCE mode thay vì GOVERNANCE cho S3 Object Lock" | ADR update + contract impact |
| #2 Medium | T2 W12 chiều (30p) | "Worker queue backlog pattern cần scale up to 3x current limit" hoặc "DynamoDB idempotency lock chuyển sang Redis" | Scale plan + ADR |
| #3 Chaos | T4 W12 chiều (60p) | "EKS control plane unreachable 30 phút" hoặc "AI engine OOM khi xử lý 5 incidents concurrent" | Failover + circuit breaker demo |

Mỗi curveball → ghi vào `curveball-responses.md` ngay.

---

## 7. Links quan trọng

- Phân tích TF chi tiết: [`01_task_force_analysis.md`](01_task_force_analysis.md)
- Build plan tuần W11: [`02_w11_build_plan.md`](02_w11_build_plan.md)
- Build plan tuần W12: [`03_w12_build_plan.md`](03_w12_build_plan.md)
- Câu hỏi cần clarify: [`04_clarification_questions.md`](04_clarification_questions.md)
- Differentiation angle analysis: [`05_differentiation_angles.md`](05_differentiation_angles.md)
- Risk registry: [`06_risk_registry.md`](06_risk_registry.md)
- Standup notes: [`standup-notes.md`](standup-notes.md)
