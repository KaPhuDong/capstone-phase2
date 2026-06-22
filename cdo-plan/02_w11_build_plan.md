# W11 Build Plan - CDO Team · TF3 Self-Heal Engine
> Tuần 22/06 - 26/06. Mục tiêu: tài liệu approve T5 + ký contract + base infra chạy được EOD T6.

---

## Tổng quan tuần

```
T2 22/06  ──  T3 23/06  ──  T4 24/06  ──  T5 25/06  ──  T6 26/06
[Kickoff]     [BA + angle]  [Doc draft]   [Onsite]      [BASE INFRA]
[Interview]   [Infra plan]  [Contract rev] [Sign]        [Pack #1 ✅]
```

**Cột mốc không dời được:**
- T4 EOD: Review contract draft từ AI, push-back ghi rõ
- T5 onsite: Ký 3 contracts
- T6 EOD: Base infra chạy được + Pack #1 submit

---

## T2 22/06 - Kickoff + Discovery

### Sáng (8h30 - 12h): Đọc TF3 Brief

**Việc phải làm ngay (cả team):**
- [ ] Đọc `TF3_SELFHEAL_LEARNER.md` **mỗi người đọc độc lập, note câu hỏi riêng**
- [ ] Focus đặc biệt vào: Hard requirements (5 safety checkpoints, ≥3 patterns, audit tamper-evident)
- [ ] Note ngay: K8s RBAC, idempotency lock, S3 Object Lock, 3 AI endpoints
- [ ] Team sync 30 phút: share câu hỏi, merge list theo 5 mảng: Cluster scope / Pattern priority / Safety / Audit / Contract
- [ ] Phân công sơ bộ: ai hỏi mảng nào trong Client interview
- [ ] Sơ bộ 1-2 differentiation angle candidate (K8s Operator vs Step Functions vs Serverless)

**Output buổi sáng:**
- List 10-15 câu hỏi Client đã deduplicate, phân mảng
- Draft sơ bộ angle preference (announce T3)

### Chiều (14h - 17h30): Client Interview

**Chuẩn bị vào phòng họp:**
- Framework hỏi: 5W2H
- 1 người ghi note, còn lại hỏi
- Capture priorities: cluster scope, pattern priority, audit requirement, blast radius, safety checkpoints

**Câu hỏi phải hỏi Client TF3 (xem đầy đủ ở `04_clarification_questions.md`):**
- K8s version + node count + namespace structure?
- 3 patterns implement đầu tiên: OOM / CrashLoop / Queue backlog - đúng không? Business impact rank?
- "Auto-resolved" = execute success, hay metric returns normal?
- Blast radius: max % pod có thể affect trong 1 action?
- Escalation policy: mấy lần thử trước escalate? Slack channel structure?
- S3 Object Lock: COMPLIANCE hay GOVERNANCE mode?
- Decision engine: pure rule-based, LLM-assisted, hay hybrid?
- Alert source: Prometheus AlertManager webhook format?

**Sau interview (ngay trong chiều):**
- [ ] Viết debrief "Tôi hiểu là..." - đặc biệt confirm: pattern list, blast radius cap, audit mode, escalation SLA
- [ ] Send debrief cho mentor confirm trước EOD

**Jira tasks T2:**
- Task: "Kickoff + đọc brief TFx" → Done + evidence: note bullet list
- Task: "Client interview + debrief" → Done + evidence: debrief doc link

---

## T3 23/06 - BA + Architecture Planning

### Mục tiêu ngày

1. Lock **differentiation angle** → announce với AI nhóm trước EOD
2. Draft `01_requirements_analysis.md` final
3. Sketch architecture sơ bộ cho `02_infra_design.md`

### Buổi sáng: Requirements Analysis

**Phân công gợi ý (team 3-4 người):**

| Người | Focus |
|-------|-------|
| CDO Lead | §1 Đề tài context + §2 NFR table + §3 Angle decision |
| Backend infra | §4 Comparison angle vs CDO khác trong TF |
| Security/Ops | §5 Constraints + §6 Open questions |
| All | Review + merge trước 12h |

**Checklist `01_requirements_analysis.md` T3:**
- [ ] §1 Đề tài context (1 paragraph, refer AI's requirements doc khi có)
- [ ] §2 NFR table điền đủ: latency target, availability, cost/tenant, onboarding SLA
- [ ] §3 Differentiation angle: chọn 1 angle, viết lý do + trade-off + locked date
- [ ] §4 Comparison: 3 columns (My angle / Nhóm khác A / Nhóm khác B) - nếu chưa biết angle kia, ghi "TBD"
- [ ] §5 Constraints đặc thù đề tài
- [ ] §6 Open questions (những gì còn cần hỏi AI nhóm)

### Buổi chiều: Architecture Sketch

**Công việc:**
- [ ] Draft Mermaid diagram cho architecture TF3 flow: alert → detect → decide → execute → verify → audit
- [ ] Component table: EKS cluster, AI engine hosting, DynamoDB lock, S3 Object Lock, alert ingestor
- [ ] Multi-tenant approach: **namespace-per-tenant** (TF3 standard, RBAC isolation)
- [ ] Viết ADR-001: Infra angle pick (K8s Operator vs Step Functions vs Serverless)
- [ ] Viết ADR-002: Compute target cho AI engine hosting (Fargate vs EKS pod vs Lambda)
- [ ] Note: announce angle cho AI nhóm **EOD T3** để AI viết Deployment contract phù hợp (cần biết: compute spec, K8s ServiceAccount setup, kubeconfig secret path)

**14h Standup (15 phút):**
- Done: interview debrief confirm
- Doing: angle locked, draft infra
- Blocker: ghi rõ nếu có

**EOD T3 announce với AI nhóm:**
- Email/Slack: "CDO [M] chọn angle: [K8s Operator / Step Functions / Serverless]. Compute target AI engine: [Fargate task / EKS pod]. Multi-tenant: namespace-per-tenant. Secrets path prefix: `tf-3/ai-engine/*`. K8s ServiceAccount name: `self-heal-engine-sa`."
- **TF3 specific**: AI nhóm cần biết CDO provision K8s RBAC như thế nào để viết Deployment contract đúng (ServiceAccount + RBAC scope + kubeconfig mount pattern).

**Jira tasks T3:**
- Task: "Draft 01_requirements_analysis.md" → evidence: commit SHA
- Task: "Architecture sketch + ADR-001, ADR-002" → evidence: commit SHA

---

## T4 24/06 - Doc Draft + Contract Review

### Mục tiêu ngày

1. Complete doc drafts đủ cho Progress #1 check (light)
2. Review contract draft từ AI, push-back documented
3. Prep co-design session T5 sáng

### Sáng: Complete Doc Drafts

**Priority order:**

1. `02_infra_design.md` (draft) - quan trọng nhất, basis cho contract discussion
   - [ ] §1 Architecture diagram (Mermaid)
   - [ ] §2 Component table
   - [ ] §3 Differentiation angle deep-dive
   - [ ] §4 Multi-tenant approach
   - [ ] §5 Alternatives considered
   - [ ] §6 Scaling strategy

2. `03_security_design.md` (skeleton) - chỉ cần §1+§2+§3+§5 cho Pack #1
   - [ ] §1 Network Security (VPC/SG diagram)
   - [ ] §2 IAM roles table
   - [ ] §3 Secrets inventory

3. `04_deployment_design.md` (sketch) - IaC strategy + pipeline stages
   - [ ] §1 IaC tool + module structure
   - [ ] §2 CI/CD pipeline stages

4. `08_adrs.md` - thêm ADR-003 cho audit storage (S3 Object Lock vs append-only DB) và ADR-004 cho idempotency lock (DynamoDB vs Redis)

### Chiều: Contract Review TF3

**EOD T4 AI publish 3 contracts draft → CDO review ngay:**

**Telemetry Contract TF3:**
- [ ] Alert payload format: Prometheus AlertManager JSON schema - CDO ingestor parse được không?
- [ ] K8s events CDO phải emit: pod state changes, metric snapshots - format và fields?
- [ ] Volume: alert frequency estimate - CDO ingestor handle được không?
- [ ] Delivery SLA: alert → AI nhận trong bao lâu?

**AI API Contract TF3 (3 endpoints):**
- [ ] `/v1/detect`: input schema - có `correlation_id` + `idempotency_key` không? CDO có đủ data từ alert?
- [ ] `/v1/decide`: output `blast_radius_ok: bool` - CDO check field này trước execute
- [ ] `/v1/decide`: output `action_plan[]` - format gì? CDO parse và execute ra sao?
- [ ] `/v1/verify`: CDO gọi sau bao nhiêu giây? Hay AI callback?
- [ ] `dry_run_mode`: boolean trong body hay header? Phải thống nhất trước khi ký
- [ ] P99 latency SLA từng endpoint - CDO timeout config phải > AI SLA mỗi endpoint
- [ ] Error codes: 429 → CDO retry với exponential backoff, 503 → circuit breaker open

**Deployment Contract TF3:**
- [ ] Compute spec (CPU/memory) - phù hợp với Fargate task hay EKS pod?
- [ ] `kubeconfig` secret: path trong Secrets Manager - CDO naming convention match không?
- [ ] K8s ServiceAccount: AI engine cần quyền gì? CDO provision RBAC đúng scope không?
- [ ] Idempotency lock: DynamoDB table name + key format - CDO tạo table đúng schema không?
- [ ] Audit storage: S3 bucket name + prefix + Object Lock mode - CDO setup đúng không?
- [ ] Network: AI engine pod cần access K8s API server - CDO SG allow port 443 không?

**Ghi push-back cụ thể:**
Mỗi push-back viết theo format:
> "Section X: [vấn đề cụ thể] → đề xuất [alternative] vì [lý do]"

**T4 cuối ngày (trước standup):**
Send push-back notes cho AI nhóm để prep co-design T5 sáng.

**Jira tasks T4:**
- Task: "Draft 02-04_infra/security/deployment docs" → evidence: commit SHA
- Task: "Review 3 contracts + push-back notes" → evidence: doc link / comment

---

## T5 25/06 - Co-Design + Onsite Approve

### Sáng (không có mentor): Co-Design với AI nhóm

**Format suggested (2-3 tiếng):**

```
[30 phút] AI walk through 3 contracts draft
[60 phút] CDO raise push-back point by point
[30 phút] Negotiate → agree final version
[30 phút] Update contracts files, review lần cuối
```

**CDO role trong co-design:**
- Đừng accept ngay - push-back productive là expected
- Mỗi push-back phải có justification kỹ thuật, không phải preference
- Document agreed changes ngay (vào contract files, không phải chat)

**Checklist contracts trước onsite:**
- [ ] Telemetry contract: finalized, cả AI + CDO đều comfortable
- [ ] AI API contract: URL fixed, schema locked, SLA agreed
- [ ] Deployment contract: compute spec, network, secrets agreed

### Chiều: Onsite Approve

**Chuẩn bị pitch ngắn cho mentor:**
- "Đây là angle chúng tôi chọn, vì X, trade-off Y, achievable trong W12 vì Z"
- Sẵn sàng giải thích architecture diagram
- Sẵn sàng defend ADR decisions

**Curveball #1 (15p) - sẵn sàng cho TF3:**
- Ví dụ: "Thêm pattern Cert expiring vào scope" → trả lời: ADR cập nhật, pattern designed-only hay implement, contract impact
- Ví dụ: "S3 Object Lock phải là COMPLIANCE mode" → trả lời: đã plan chưa, terraform change gì, cost impact
- Ví dụ: "DynamoDB lock chuyển sang Redis" → trả lời: Deployment contract impact, Terraform module nào thay đổi
- Viết response vào `curveball-responses.md` ngay sau buổi

**EOD T5:**
- [ ] 3 contracts ký xong (**FROZEN từ đây**)
- [ ] Mentor approve bộ doc

**Jira tasks T5:**
- Task: "Co-design contracts với AI nhóm" → evidence: final contract commit SHA
- Task: "Onsite approve + curveball #1 response" → evidence: curveball-responses.md commit

---

## T6 26/06 - Base Infra Sprint

### Mục tiêu ngày: BASE INFRA CHẠY ĐƯỢC trước EOD

Đây là ngày build duy nhất của W11. Phân công rõ ràng, parallel execution.

### Ưu tiên tuyệt đối (must-have cho Pack #1)

```
Priority 1: VPC + networking + SG + VPC endpoints (30-45 phút)
Priority 2: EKS cluster + node group + OIDC provider (1-2 giờ) ← TF3 khác các TF khác!
Priority 3: K8s RBAC: namespace-per-tenant, ServiceAccount, ClusterRole
Priority 4: AI skeleton endpoint accessible từ CDO (Fargate hoặc EKS pod)
Priority 5: S3 Object Lock bucket + DynamoDB idempotency table (audit infra)
Priority 6: Basic alert ingestor: API GW → Lambda nhận webhook → log
Priority 7: Smoke test: alert webhook → ingest → /v1/detect skeleton call → 200 + dummy JSON
```

> **TF3 khác TF1/TF2**: base infra phức tạp hơn vì có EKS cluster + K8s RBAC setup. Cần ~2 ngày thay vì ~1 ngày.

### Phân công gợi ý T6 TF3

| Track | Ai | Task |
|-------|----|------|
| **EKS + RBAC** | Infra Lead | Terraform: EKS cluster, node group, OIDC, namespace, RBAC |
| **Networking + IAM** | Infra 2 | VPC, subnets, SG, VPC endpoints, IAM roles (task role, deploy role) |
| **Audit infra** | Backend | S3 Object Lock, DynamoDB idempotency table, Athena workgroup, Glue |
| **Alert ingestor + AI connect** | Backend | API GW + Lambda ingestor, AI skeleton call, smoke test |
| **Doc finalize** | Lead | Pack #1 docs commit, ADRs, standup notes |

### Terraform module structure TF3

```
infra/
├── modules/
│   ├── networking/         # VPC, subnets, SG, VPC endpoints (Bedrock, SM, S3)
│   ├── eks-cluster/        # EKS cluster, managed node group, OIDC provider
│   ├── eks-rbac/           # Namespace, ServiceAccount, ClusterRole, RoleBinding
│   ├── ai-engine/          # Fargate task / EKS pod for AI engine skeleton
│   ├── audit-storage/      # S3 bucket (Object Lock), Glue crawler, Athena workgroup
│   ├── idempotency-lock/   # DynamoDB table (hash key: idempotency_key)
│   ├── alert-ingestor/     # API GW + Lambda (receive AlertManager webhook)
│   ├── observability/      # CloudWatch / Prometheus + Grafana
│   └── tenant/             # Per-tenant: K8s namespace + RBAC (reusable)
├── environments/
│   └── sandbox/
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfvars
└── README.md
```

### Definition of Done - Base Infra T6 (TF3)

- [ ] `terraform apply` chạy clean
- [ ] EKS cluster healthy: `kubectl get nodes` → Ready
- [ ] ≥2 tenant namespaces created với RBAC (namespace `tenant-a`, `tenant-b`)
- [ ] ServiceAccount `self-heal-engine-sa` created với RBAC role (không có cluster-admin!)
- [ ] S3 Object Lock bucket exists + GOVERNANCE/COMPLIANCE mode configured
- [ ] DynamoDB table `idempotency-lock` exists với partition key `idempotency_key`
- [ ] AI skeleton endpoint accessible: `POST /v1/detect` → `{ "anomaly_type": "OOMKilled", "confidence": 0.85, "correlation_id": "..." }`
- [ ] Alert ingestor: `POST /webhook/alert` → 200 OK + log
- [ ] README trong `infra/` mô tả cách run + kubectl access setup
- [ ] Commit history: ≥8 commits với message rõ ràng (không 1 fat commit)

### EOD T6 - Pack #1 Submit

**Checklist cuối ngày:**
- [ ] `docs/01_requirements_analysis.md` ✅ (final)
- [ ] `docs/02_infra_design.md` ✅
- [ ] `docs/03_security_design.md` (draft OK)
- [ ] `docs/04_deployment_design.md` (draft OK)
- [ ] `docs/05_cost_analysis.md` (skeleton với forecast)
- [ ] `docs/08_adrs.md` ≥3 ADRs
- [ ] `infra/` running + smoke test passed
- [ ] `standup-notes.md` updated
- [ ] Jira: tất cả task Done có evidence link

**Jira tasks T6:**
- Task: "Terraform EKS cluster + node group" → evidence: commit SHA + `kubectl get nodes` output
- Task: "K8s RBAC: namespaces + ServiceAccount + ClusterRole" → evidence: `kubectl describe sa self-heal-engine-sa`
- Task: "Audit infra: S3 Object Lock + DynamoDB table" → evidence: commit SHA + AWS console screenshot
- Task: "Alert ingestor Lambda + smoke test" → evidence: CloudWatch log showing webhook received
- Task: "AI skeleton 3 endpoints integrated" → evidence: screenshot response JSON từ /v1/detect
- Task: "Finalize Pack #1 docs" → evidence: commit SHA
