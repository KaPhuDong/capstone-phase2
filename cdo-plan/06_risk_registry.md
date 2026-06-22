# Risk Registry - CDO Team · TF3 Self-Heal Engine
> Rủi ro đã biết, dấu hiệu cảnh báo, và plan mitigation.
> Cập nhật khi phát hiện risk mới hoặc khi risk thay đổi severity.

---

## Khung đánh giá risk

```
Severity = Likelihood × Impact
Likelihood: Low / Medium / High
Impact:     Low / Medium / High / Critical
```

| Level | Hành động |
|-------|----------|
| 🔴 Critical | Escalate mentor ngay, không chờ standup |
| 🟠 High | Discuss tại standup hôm nay |
| 🟡 Medium | Track + mitigation plan ready |
| 🟢 Low | Monitor, không cần action ngay |

---

## Risk W11 - Tuần tài liệu

### R01 - Scope Misunderstanding sau Client Interview 🟠

**Risk**: Hiểu nhầm requirement Client → build wrong thing → rework T3-T4 mất 2 ngày

**Dấu hiệu**: 
- Debrief "tôi hiểu là..." không được mentor confirm
- Architecture sketch bị AI nhóm phản bác vì không match đề tài

**Mitigation**:
- Viết debrief ngay EOD T2, send mentor confirm ngay
- Cross-check với file `TFx_*_LEARNER.md` - mọi assumption phải có source
- Câu hỏi "Out of scope" hỏi rõ với Client - cái gì NOT build

**Owner**: CDO Lead
**Deadline**: EOD T2 W11

---

### R02 - AI và CDO Không Thống Nhất Contract 🟠

**Risk**: AI publish contract draft T4, CDO reject nhiều điểm, không kịp co-design T5 sáng

**Dấu hiệu**:
- Contract draft AI publish muộn (sau 17h T4)
- CDO push-back nhiều điểm fundamental (không phải tweaks)
- Không đồng ý được về latency SLA hoặc schema

**Mitigation**:
- CDO announce angle + expected compute spec sớm (T3 W11) để AI biết viết contract phù hợp
- Review contract ngay khi AI publish, push-back note gửi trong đêm T4
- T5 sáng co-design: focus vào 3-4 điểm critical nhất, đừng bike-shed minor details
- Last resort: accept AI's version nếu CDO truly can implement, document concern trong ADR

**Owner**: CDO Lead + AI Liaison
**Deadline**: EOD T4 W11

---

### R03 - EKS Cluster Không Ready EOD T6 🔴

**Risk**: TF3 cần EKS cluster (không phải chỉ Lambda/Fargate). EKS provision lâu hơn (~15-20 phút) và nhiều configuration hơn. Nếu stuck → fail Pack #1 base infra.

**Dấu hiệu**:
- 12h T6 `terraform apply` cho EKS module chưa xong
- `kubectl get nodes` trả Not Ready hoặc error
- OIDC provider chưa configured → IRSA fail → AI engine không có AWS access

**Mitigation**:
- T5 onsite chiều: sau ký contract, bắt đầu `terraform apply` EKS ngay (EKS provision = blocking)
- EKS trước, mọi thứ khác sau
- "Break glass" nếu EKS stuck: có thể dùng Fargate-only (ECS) để demo AI engine integration, EKS setup tiếp T6
- T6 8h: EKS module phải đang apply hoặc đã done, không bắt đầu sau 10h

**Owner**: Infra Lead
**Escalate nếu**: 14h T6 `kubectl get nodes` chưa Ready → ping mentor ngay

---

### R04 - Angle Overlap với CDO Kia Trong Task Force 🟡

**Risk**: Cả 2 CDO chọn cùng angle → không có differentiation → cap Tier 2

**Dấu hiệu**:
- CDO kia announce angle T3 W11, trùng với angle của mình
- T5 W11 mentor hỏi "hai bạn CDO khác nhau gì?"

**Mitigation**:
- Announce angle sớm nhất có thể (T3 sáng) để CDO kia biết mà chọn khác
- Nếu overlap: agree ngay T3 ai switch → đừng để muộn đến T5
- Fallback: cùng angle nhưng khác implementation layer (vd: cả 2 Fargate nhưng 1 ECS-standard, 1 ECS+App Mesh) → khó differentiate, nên tránh

**Owner**: CDO Lead
**Deadline**: T3 W11 EOD

---

## Risk W12 - Tuần build

### R05 - AI Engine 3 Endpoints Không Work T3 Integration 🔴

**Risk**: TF3 có 3 endpoints (detect/decide/verify). Nếu AI engine partial - ví dụ detect OK nhưng decide broken - CDO vẫn có gap lớn trong E2E flow.

**Dấu hiệu**:
- `/v1/detect` OK nhưng `/v1/decide` trả sai `action_plan` format
- `/v1/verify` không trả `regression` khi test rollback scenario
- Response không có `blast_radius_ok` field → CDO decision gate fail

**Mitigation**:
- CDO maintain mock server cho tất cả 3 endpoints, có thể bật lại ngay
- Document gap cụ thể per endpoint: "detect OK, decide partial (thiếu field X), verify not tested"
- Tiếp tục E2E với mock cho endpoint bị lỗi
- Escalate mentor + AI team lead sau 30 phút debug không resolve

**Owner**: CDO Integration lead
**Escalate nếu**: bất kỳ endpoint nào vẫn sai schema sau 30 phút debug

---

### R06 - Member Ôm Hết Task (Free Rider Risk) 🟠

**Risk**: 1-2 người làm hết, còn lại không có evidence → individual defense fail → cap Tier 3

**Dấu hiệu** (per playbook):
- 1 người > 60% commits trong repo
- 1 member < 5 Jira tasks Done trong 2 tuần
- Task của 1 người chỉ là "standup note" / "doc edit" không có real work

**Mitigation**:
- T2 W12: CDO lead review Jira task assignment, ensure balanced
- Mỗi người phải own ít nhất 1 infra module hoặc 1 integration flow
- Daily standup: mỗi người report cụ thể task đang làm (không chỉ "đang review")
- Nếu phát hiện: redistribute tasks ngay, không chờ mentor sample Jira

**Owner**: CDO Lead
**Check point**: EOD T2 W12

---

### R07 - Curveball #3 Phá Infra 🟠

**Risk**: T4 W12 curveball chaos inject scenario CDO không handle được → infra broken → không có demo T5

**Dấu hiệu**:
- `terraform apply` fail sau curveball
- Service down, không recover được trong 60p
- Rollback cũng fail

**Mitigation**:
- T3 W12 EOD: git tag `stable-pre-chaos` để có clean rollback point
- Curveball response process: 10p assess → 25p execute → verify trước commit
- "Do no harm" rule: nếu fix risk làm hỏng thêm thứ khác → rollback về `stable-pre-chaos`
- Dry-run T4 17h: run demo với state sau curveball, không run với state trước

**Owner**: Infra Lead
**Backup plan**: nếu curveball phá demo, show recording pre-curveball + document curveball response

---

### R08 - Slide Đẹp Nhưng Q&A Fail 🟡

**Risk**: Team làm slide đẹp nhưng individual member không walk-through được commit → cap Tier 3

**Dấu hiệu**:
- Mock Q&A T4 W12: member không trả lời được "sao chọn X?"
- Member không nhớ commit SHA cho task mình Done
- Member nói "người kia làm, tôi không biết"

**Mitigation**:
- T3 W12: mỗi người tự review lại Jira task của mình, chuẩn bị walk-through
- T4 W12 17h dry-run: practice "random pick" individual defense (không phải chỉ lead present)
- Format chuẩn bị: "Task X: tôi làm gì, file Y, commit Z, decision tôi đưa ra vì reason W"
- Mọi người phải có ít nhất 1 ADR decision có thể defend

**Owner**: Tất cả members
**Practice**: T4 W12 dry-run 17h-18h

---

### R09 - Cost Vượt Budget 🟡

**Risk**: AWS bill spike trong W12 (EKS node, RDS, load test) → account block hoặc overspend

**Dấu hiệu**:
- Daily cost spike unexpected
- Load test chạy quá mạnh
- Forgot to delete resources after test

**Mitigation**:
- Setup AWS Budget alert: 50% / 80% / 100% của budget cap → Slack/email notify
- Load test: có hard limit (duration + RPS cap)
- After each test: run cleanup script
- T4 W12 EOD: run `terraform destroy` cho tất cả non-essential resources trước code freeze

**Owner**: Ops member
**Alert threshold**: Ngay khi nhận 80% budget alert

---

### R10 - Doc Viết Một Phát Cuối (Git History Thin) 🟡

**Risk**: Viết toàn bộ doc ngày T6 W11 / T4 W12 → git history = 1-2 fat commit → evidence process thinking kém

**Dấu hiệu**:
- `git log` chỉ có 1-2 commit cho cả folder docs
- Commit message chung chung: "add docs" / "update files"

**Mitigation**:
- Rule: mỗi ngày ít nhất 1 doc commit với message cụ thể
- Viết doc song song với build, không để đến cuối
- Commit convention: `docs(02_infra): add multi-tenant isolation section`
- Review git log T4 W12 trước submit: phải thấy commits spread across W11-W12

**Owner**: Tất cả members
**Check point**: T3 W11, T3 W12

---

### R11 - Auto-Resolve Rate < 60% 🟠

**Risk**: TF3 hard requirement: ≥60% trên ≥10 scenarios. Nếu CDO decision gate, K8s action execution, hoặc verify logic có bug → rate thấp → fail rubric.

**Dấu hiệu**:
- T3 W12 scenario simulation: chạy S1-S6 thấy >40% fail
- K8s action execute xong nhưng pod không healthy → verify trả regression
- CDO circuit breaker open quá sớm (threshold quá thấp) → quá nhiều escalate

**Mitigation**:
- T2 W12: test K8s action execution độc lập (không qua AI) để verify CDO K8s code works
- Calibrate circuit breaker threshold với Client input (T2 W11)
- Nếu rate thấp T3 sáng: debug pattern-by-pattern, identify which pattern fail most
- Document honestly: "tôi đạt X%, gap là Y do Z"

**Owner**: Backend Engineer + CDO Lead
**Check point**: T3 W12 sáng sau S1-S6

---

### R12 - RBAC Quá Rộng / Quá Hẹp 🟠

**Risk**: K8s ServiceAccount cho AI engine có quyền cluster-admin (unsafe action risk) hoặc quá hẹp (engine fail lúc execute action).

**Dấu hiệu**:
- `kubectl auth can-i patch pods --as=system:serviceaccount:default:self-heal-engine-sa` trả YES cho namespace khác
- `/v1/decide` action execute fail với Forbidden error

**Mitigation**:
- T6 W11: test RBAC ngay sau setup: `kubectl auth can-i` per namespace per verb
- Principle: ServiceAccount chỉ có get/list/patch/update trong namespace app workload, không có delete
- Verify tenant isolation: self-heal-engine-sa của tenant-a không access namespace tenant-b
- Security scan T4 W12: check RBAC là một phần

**Owner**: Security member
**Check point**: T6 W11 sau RBAC setup + T3 W12 isolation test

---

## Risk Dashboard Summary (Updated TF3)

| Risk | Level | Owner | Status |
|------|-------|-------|--------|
| R01 Scope misunderstanding | 🟠 High | CDO Lead | ⬜ Open |
| R02 Contract disagreement | 🟠 High | CDO Lead + AI | ⬜ Open |
| R03 EKS cluster không ready T6 | 🔴 Critical | Infra Lead | ⬜ Open |
| R04 Angle overlap | 🟡 Medium | CDO Lead | ⬜ Open |
| R05 AI 3 endpoints fail T3 | 🔴 Critical | Integration Lead | ⬜ Open |
| R06 Free rider | 🟠 High | CDO Lead | ⬜ Open |
| R07 Curveball phá infra | 🟠 High | Infra Lead | ⬜ Open |
| R08 Q&A fail | 🟡 Medium | All members | ⬜ Open |
| R09 Cost spike (EKS node) | 🟡 Medium | Ops member | ⬜ Open |
| R10 Thin git history | 🟡 Medium | All members | ⬜ Open |
| R11 Auto-resolve rate < 60% | 🟠 High | Backend + Lead | ⬜ Open |
| R12 RBAC too wide/narrow | 🟠 High | Security | ⬜ Open |
