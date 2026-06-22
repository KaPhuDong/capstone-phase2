# Clarification Questions - CDO Team
> Câu hỏi cần làm rõ: với Client (T2 W11), với AI nhóm (T3-T4 W11), và assumptions chưa giải quyết.
> Cập nhật status khi có answer.

---

## Framework hỏi Client: 5W2H

Khi vào phòng họp T2 chiều, mỗi câu hỏi phải thuộc 1 trong 7 nhóm:

| Nhóm | Mục đích |
|------|---------|
| **What** | Scope rõ ràng - cái gì IN / OUT |
| **Why** | Priority + business driver |
| **Who** | Stakeholder, team owner, user persona |
| **When** | Timeline, SLA, cadence |
| **Where** | Deployment region, data residency |
| **How** | Process, workflow, integration |
| **How much** | Budget, scale, threshold |

Đừng hỏi "Anh muốn gì?" - đó là câu tệ nhất. Hỏi specific.

---

## Câu hỏi chung (áp dụng mọi TF)

### Scope + Priority

- [ ] **What**: Đâu là 1 thứ quan trọng nhất nếu demo phải chọn? (tránh scope creep)
- [ ] **What**: Feature nào là must-have vs nice-to-have?
- [ ] **What**: Có constraint nào chưa được mention trong brief không?
- [ ] **Why**: Business metric nào Client dùng để đo success? (không phải "tôi thích")

### Technical Environment

- [ ] **Where**: AWS region nào? Có dùng ap-southeast-1 hay us-east-1?
- [ ] **How**: AWS account setup: fresh account hay reuse? Organization hay standalone?
- [ ] **How**: Existing tooling: monitoring stack hiện tại là gì? (CloudWatch / Prometheus / Datadog?)
- [ ] **How**: Authentication: IAM-only hay có external IdP (Cognito / Okta)?
- [ ] **How much**: AWS budget cho capstone (không muốn overspend)?

### Multi-tenant

- [ ] **Who**: Tenant là gì trong context này? (user / team / company / AWS account?)
- [ ] **How much**: Số tenant trong demo? 3? 10? 50?
- [ ] **What**: Tenant isolation requirement: data chỉ hay cả compute?
- [ ] **When**: Onboarding SLA: tenant mới ready trong bao lâu là acceptable?

### Compliance + Security

- [ ] **What**: SOC2 controls cụ thể nào áp dụng? Type I hay Type II?
- [ ] **How long**: Audit log retention: 90 ngày hay lâu hơn?
- [ ] **What**: PII trong payload có không? Nếu có: redact hay không ingest?
- [ ] **What**: Data residency hard requirement không?

### Failure + SLA

- [ ] **What**: Khi AI engine down, behavior expected là gì? (fail-open / fail-closed / fallback)
- [ ] **How much**: Availability target cho CDO platform: 99.5% / 99.9%?
- [ ] **When**: P99 latency budget: bao nhiêu ms từ request → response?
- [ ] **What**: Rollback strategy: CDO tự rollback hay phải approval?

---

## Câu hỏi TF3 - Self-Heal Engine

### Với Client (T2 W11 chiều)

#### Mảng 1: Cluster + Workload Scope

- [ ] **What**: EKS cluster version? Node type (t3.medium / m5.large)? Node count?
- [ ] **What**: Namespace structure hiện tại trong production? (tham khảo để design sandbox)
- [ ] **How much**: Khoảng bao nhiêu pod total trong sandbox? Để CDO estimate blast radius.
- [ ] **What**: 3 patterns ưu tiên nhất: OOM / CrashLoop / Queue backlog - đúng không? Rank theo business impact?
- [ ] **What**: "Auto-resolved" = action execute thành công, hay metric thật sự returns normal?

#### Mảng 2: Safety + Blast Radius

- [ ] **How much**: Blast radius hard cap: tối đa bao nhiêu % pod trong 1 namespace có thể affect trong 1 action?
- [ ] **How much**: Circuit breaker threshold: mấy lần consecutive failure trước khi circuit open?
- [ ] **When**: Escalation policy: mấy lần thử trước escalate? (1 lần / 3 lần / per-pattern)?
- [ ] **When**: Escalation SLA: engineer phải respond trong bao lâu sau message?
- [ ] **What**: Rollback method per pattern: Git SHA revert / resource snapshot / kubectl apply previous state?

#### Mảng 3: Audit + Compliance

- [ ] **What**: S3 Object Lock mode: COMPLIANCE (không ai delete, kể cả root) hay GOVERNANCE (admin có thể override)?
- [ ] **How long**: Audit log retention: 90 ngày là minimum, có cần lâu hơn không?
- [ ] **Who**: Ai có quyền query audit log? (SRE only / Engineering manager / External auditor?)
- [ ] **What**: Audit format: JSON per-event hay batch? Fields bắt buộc nào?
- [ ] **What**: SOC2 Type II controls cụ thể: CC6.1 (access control) / CC7.2 (monitoring) - controls nào liên quan trực tiếp đến audit?

#### Mảng 4: Alert Source + Integration

- [ ] **What**: Alert source: Prometheus AlertManager webhook format - standard JSON hay custom?
- [ ] **How**: AlertManager webhook route đến đâu? CDO cần setup receiver URL như thế nào?
- [ ] **What**: Escalation notification: Slack webhook channel structure? Bot đã có hay cần tạo?
- [ ] **What**: Khi AI engine down, hành vi expected: queue incidents và retry, hay immediately escalate, hay silent fail?

#### Mảng 5: Decision Engine + Scope

- [ ] **How**: Decision engine preference: pure rule-based / LLM-assisted / hybrid? (ảnh hưởng CDO contract expectations)
- [ ] **What**: Sandbox workload inject: CDO tự build script hay mentor cung cấp tool?
- [ ] **When**: Latency budget: detect → action bao nhiêu ms? action → verify bao nhiêu ms?
- [ ] **What**: Có pattern nào NEVER auto-execute (chỉ escalate ngay)? (ngoài NEVER list đã có)
- [ ] **What**: `dry_run_mode`: default ON hay OFF khi CDO platform mới deploy?

---

### Với AI nhóm TF3 (T3-T4 W11)

#### Về 3 Endpoints

- [ ] `/v1/detect` input schema: mandatory fields? `idempotency_key` CDO generate hay AI generate?
- [ ] `/v1/detect` output: `anomaly_type` enum values? `confidence` float 0-1?
- [ ] `/v1/decide` input: CDO pass `/v1/detect` output trực tiếp hay transform?
- [ ] `/v1/decide` output `action_plan[]`: format gì? (kubectl command string? structured action object?)
- [ ] `/v1/decide` output `blast_radius_ok`: boolean hay có threshold float?
- [ ] `/v1/verify` input: CDO phải gửi gì? (correlation_id + current pod state?)
- [ ] `/v1/verify` output `status` enum: `success` / `regression` / `next_action` - `next_action` có payload không?
- [ ] Optional `/v1/rollback`: có build không? CDO cần plan cho endpoint này không?

#### Về Contract Fields

- [ ] `dry_run_mode`: boolean trong request body hay HTTP header? → phải nhất quán trong contract
- [ ] `correlation_id`: CDO generate từ alert ID (UUID v4)? AI echo back trong mọi response?
- [ ] `idempotency_key`: format? CDO generate SHA256(alert_id + timestamp_rounded_to_5min)?
- [ ] P99 latency SLA từng endpoint riêng? (detect thường nhanh hơn decide)
- [ ] Error code 429: retry-after header có không? CDO cần implement exponential backoff?

#### Về Deployment Contract

- [ ] K8s ServiceAccount name: `self-heal-engine-sa` hay AI đặt tên khác?
- [ ] RBAC scope AI engine cần: get/list/patch pods, rollout restart deployments, scale deployments - đủ không?
- [ ] `kubeconfig` secret: AI engine mount in-cluster ServiceAccount hay cần full kubeconfig file?
- [ ] Idempotency lock: DynamoDB table name + partition key format?
- [ ] Audit S3 bucket: naming convention? Prefix per tenant?
- [ ] Compute spec: vCPU/memory cho AI engine Fargate task hoặc EKS pod?

---

## Assumption Log TF3

| # | Assumption | Risk nếu sai | Confirm với | Status |
|---|-----------|-------------|-------------|--------|
| A1 | AWS region us-east-1 (brief đã mention) | IaC wrong region | Client T2 | ⬜ Open |
| A2 | EKS K8s version 1.29+ | Terraform EKS module version mismatch | Client T2 | ⬜ Open |
| A3 | Prometheus AlertManager là alert source | Alert ingestor design wrong | Client T2 | ⬜ Open |
| A4 | S3 Object Lock GOVERNANCE mode (có thể delete với root) | Compliance audit fail | Client T2 | ⬜ Open |
| A5 | `dry_run_mode` là boolean field trong request body | Contract mismatch khi ký | AI nhóm T3 | ⬜ Open |
| A6 | CDO generate `idempotency_key` = UUID v4 từ alert ID | Double-execute không detect được | AI nhóm T3 | ⬜ Open |
| A7 | Namespace-per-tenant là đủ cho RBAC isolation (không cần cluster-per-tenant) | Isolation audit fail | Client T2 | ⬜ Open |
| A8 | 3 patterns để implement: OOMKilled, CrashLoopBackOff, Queue backlog | Wrong pattern prioritization | Client T2 | ⬜ Open |
| A9 | Circuit breaker threshold = 3 consecutive failures | Too tight / too loose | Client T2 | ⬜ Open |
| A10 | Terraform là IaC tool acceptable | Phải switch tool | Mentor T2 | ⬜ Open |

**Cập nhật Status:**
- ⬜ Open → 🟡 In progress → ✅ Confirmed / ❌ Wrong-assumption-corrected

---

## Questions CDO push-back với AI nhóm về Contracts

### Những gì CDO có quyền yêu cầu thay đổi

1. **Latency SLA quá tight**: Nếu AI đề xuất P99 < 100ms nhưng CDO platform có cold start → push-back "P99 <500ms feasible hơn, vì [lý do]"

2. **Telemetry volume quá cao**: Nếu AI muốn CDO emit 10 fields mỗi event nhưng CDO log pipeline chỉ hỗ trợ 5 → push-back + propose compromise

3. **Deployment compute spec quá lớn**: AI propose 8 vCPU / 16GB nhưng capstone budget không đủ → push-back "4 vCPU / 8GB đủ cho demo scale"

4. **Network requirement mâu thuẫn**: AI muốn public endpoint nhưng security design yêu cầu private → cần negotiate rõ

5. **Secrets path format không match**: AI propose `prod/ai-engine/*` nhưng CDO plan `tf-<N>/ai-engine/*` → nhỏ nhưng phải thống nhất trước khi ký

### Những gì CDO KHÔNG nên push-back

- Input/output schema logic (đó là AI domain)
- Model choice (Bedrock model nào)
- Eval metrics threshold
- Prompt engineering approach
