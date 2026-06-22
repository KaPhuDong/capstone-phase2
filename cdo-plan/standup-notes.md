# Standup Notes - CDO Team · TF3 Self-Heal Engine
> Append-only log. KHÔNG xóa entry cũ.
> Format: 14h mỗi ngày, 15 phút strict, per task force (AI lead + CDO leads).
> Mỗi người ≤30 giây: **Done / Doing / Blocker**.
> Log xong → commit ngay (evidence cho process).

---

## Template entry

```markdown
## [Ngày DD/MM] - Standup 14:00

**[Tên - Role]**
- Done: [task cụ thể, link Jira ID nếu có]
- Doing: [đang làm gì]
- Blocker: [nếu có - mô tả cụ thể]

**[Tên - Role]**
- Done:
- Doing:
- Blocker:

Task force status: 🟢 Green / 🟡 Yellow / 🔴 Red
Action items:
- [ ] [action - owner - deadline]
```

---

## Red flags tự escalate ngay (không chờ standup hôm sau)

Theo brief mentor, 3 dấu hiệu phải ping mentor ngay:
1. **2 ngày liên tiếp** cùng 1 blocker chưa resolve
2. **AI và CDO disagree** contract interpretation, không thống nhất được
3. **Build progress < 50%** expected mid-week

TF3-specific thêm:
- EKS cluster chưa Ready sau 16h T6 W11 → escalate
- 3 AI endpoints không work sau 30p debug T3 W12 → escalate
- Auto-resolve rate < 40% sau S1-S6 → escalate

---

## W11 - Tuần tài liệu + Base Infra

### W11 T2 22/06 - Standup 14:00

> _[Fill in sau standup - append tại đây]_
>
> Gợi ý check: debrief "tôi hiểu là..." đã gửi mentor chưa? Angle candidate nào?

---

### W11 T3 23/06 - Standup 14:00

> _[Append here]_
>
> Gợi ý check: angle locked chưa? Đã announce cho AI nhóm chưa? (K8s Operator / Step Functions / Serverless)

---

### W11 T4 24/06 - Standup 14:00

> _[Append here]_
>
> Gợi ý check: `02_infra_design.md` draft có diagram chưa? Contract draft từ AI publish chưa? Push-back notes ready?

---

### W11 T5 25/06 - Standup 14:00 (onsite)

> _[Append here]_
>
> Gợi ý check: 3 contracts ký xong chưa? Mentor approve chưa? EKS terraform đã bắt đầu chưa?

---

### W11 T6 26/06 - Standup 14:00

> _[Append here]_
>
> Gợi ý check: `kubectl get nodes` Ready? RBAC setup xong? S3 Object Lock bucket tồn tại? AI skeleton 3 endpoints respond?

---

## W12 - Tuần build + integrate

### W12 T2 29/06 - Standup 14:00

> _[Append here]_
>
> Gợi ý check: CI/CD pipeline green? 5 safety checkpoints code xong mấy cái? AI skeleton vẫn accessible? Curveball #2 response documented?

---

### W12 T3 30/06 - Standup 14:00

> _[Append here]_
>
> Gợi ý check: S1-S6 scenarios chạy xong? Auto-resolve rate hiện tại? Integration session 14h: 3 endpoints real pass?

---

### W12 T4 01/07 - Standup 14:00

> _[Append here]_
>
> Gợi ý check: Athena audit query works? `07_test_eval_report.md` đang viết? Curveball #3 14h: đã prep?

---

<!-- Sau code freeze T4 18h: không thêm build entry, chỉ slide/script notes nếu cần -->
