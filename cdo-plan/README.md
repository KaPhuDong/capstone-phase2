# CDO Team - Planning Hub · TF3 Self-Heal Engine
> Capstone Phase 2 · W11-W12 (22/06 - 03/07/2026)
> Team role: Cloud/DevOps · Task Force 3 · Build infra platform hosting Self-Heal Engine

---

## Đọc theo thứ tự này

| Bước | File | Khi nào đọc |
|------|------|-------------|
| 1 | [`00_PROJECT_OVERVIEW.md`](00_PROJECT_OVERVIEW.md) | **Ngay bây giờ** - tổng quan TF3, timeline, scoring |
| 2 | [`01_task_force_analysis.md`](01_task_force_analysis.md) | Trước T2 W11 - TF3 deep analysis, 3 candidate angles, 12 scenarios |
| 3 | [`05_differentiation_angles.md`](05_differentiation_angles.md) | Trước T3 W11 - K8s Operator vs Step Functions vs Serverless |
| 4 | [`04_clarification_questions.md`](04_clarification_questions.md) | T2 W11 - câu hỏi Client + AI nhóm TF3 |
| 5 | [`02_w11_build_plan.md`](02_w11_build_plan.md) | Daily W11 - checklist từng ngày |
| 6 | [`03_w12_build_plan.md`](03_w12_build_plan.md) | Daily W12 - scenario simulation + integration |
| 7 | [`06_risk_registry.md`](06_risk_registry.md) | Khi gặp vấn đề - 12 risks TF3-specific |
| 8 | [`07_jira_task_breakdown.md`](07_jira_task_breakdown.md) | T2 W11 - 40 tasks sẵn sàng pull vào Jira |

---

## Working files (cập nhật liên tục)

- [`standup-notes.md`](standup-notes.md) - append-only daily standup log, 14h mỗi ngày
- `curveball-responses.md` - tạo sau curveball #1 (T5 W11 onsite)
- `test_simulation_log.md` - tạo T3 W12, log 12 scenario results + auto-resolve rate

---

## Quick reference - Dates không được miss

```
T2 22/06  Đọc TF3 brief + chuẩn bị 15 câu hỏi + Client interview chiều
T3 23/06  Lock angle (K8s Operator / Step Functions / Serverless) → announce AI nhóm
           Announce: angle + compute target + ServiceAccount name + secrets path prefix
T4 24/06  EOD: review 3 contract drafts từ AI (3 endpoints: detect/decide/verify) + push-back
T5 25/06  Sáng: co-design contracts. Onsite: approve doc + KÝ 3 CONTRACTS (frozen)
           Ngay sau ký: bắt đầu terraform EKS (EKS provision ~15-20 phút)
T6 26/06  EOD: EKS running + RBAC setup + S3 Object Lock + DynamoDB + skeleton endpoint

T2 29/06  5 safety checkpoints code + observability. 16h: Curveball #2
T3 30/06  Sáng: inject S1-S12 scenario simulation (≥4h window)
           14h-16h: INTEGRATION SESSION (call 3 AI endpoints thật)
T4 01/07  Polish. 14h: Curveball #3 chaos. 18h: CODE FREEZE + Pack #2 submit
T5 02/07  8h: Hard freeze. Onsite: PITCH + INDIVIDUAL DEFENSE
           TF3 pitch slot: 13h30-15h15
```

---

## TF3 Hard Requirements - checklist nhanh

Phải có trước code freeze T4 W12:

```
[ ] ≥3 patterns implemented + tested (OOMKilled, CrashLoop, QueueBacklog)
[ ] ≥2 patterns designed-only (paper playbook + diagram + ADR)
[ ] Auto-resolve rate ≥60% trên ≥10 scenarios
[ ] Scenario simulation ≥4h window
[ ] 5 safety checkpoints đều demo được: dry-run, blast-radius, verify, rollback, circuit breaker
[ ] Zero unsafe action (RBAC least-privilege, no cluster-admin)
[ ] Audit log tamper-evident (S3 Object Lock) + queryable (Athena)
[ ] Multi-tenant ≥2 tenants RBAC isolation
[ ] Escalation message AI-generated với context bundle
[ ] ≥5 ADRs (brief TF3 yêu cầu explicitly)
[ ] 3 contracts ký, frozen T5 W11
```

---

## Cấu trúc folder khi build thật (TF3)

```
capstone/
└── tf-3/
    └── cdo-<M>/
        ├── docs/
        │   ├── 01_requirements_analysis.md
        │   ├── 02_infra_design.md
        │   ├── 03_security_design.md
        │   ├── 04_deployment_design.md
        │   ├── 05_cost_analysis.md
        │   ├── 07_test_eval_report.md
        │   ├── 08_adrs.md          ← ≥5 ADRs (TF3 brief requirement)
        │   └── assets/             ← diagrams, screenshots
        ├── infra/                  ← Terraform modules
        │   ├── modules/
        │   │   ├── networking/
        │   │   ├── eks-cluster/
        │   │   ├── eks-rbac/
        │   │   ├── ai-engine/
        │   │   ├── audit-storage/  ← S3 Object Lock + Glue + Athena
        │   │   ├── idempotency-lock/ ← DynamoDB
        │   │   ├── alert-ingestor/
        │   │   └── observability/
        │   └── environments/sandbox/
        ├── manifests/              ← K8s manifests (nếu K8s Operator angle)
        │   ├── crds/
        │   ├── controller/
        │   └── tenants/
        ├── scripts/
        │   └── inject_incident.sh  ← scenario simulation tool
        ├── standup-notes.md
        ├── curveball-responses.md
        ├── test_simulation_log.md
        ├── individual-pitches.md
        ├── retrospective.md
        ├── SLIDES.pdf
        └── demo-video.mp4
```

Copy templates từ `templates/cdo/` vào `docs/`, fill in theo guidance comments.
