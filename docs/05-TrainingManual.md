# 05 — Training Manual

Role-based workflow walkthroughs for the 13 personas the system serves. Each section maps to an agent definition under [.claude/agents/](../agents/) and to permissions in [08-SecurityGuide.md](08-SecurityGuide.md).

**See also:** [08-SecurityGuide.md](08-SecurityGuide.md) · [12-BusinessRules.md](12-BusinessRules.md) · [03-ApiReference.md](03-ApiReference.md)

> **Note on UI screenshots:** Screens for each workflow are **Not yet implemented**. This manual describes the intended task flow at a functional level. Update with screenshots as the Next.js and React Native clients land.

---

## Leadership

### school_head

Agent: [.claude/agents/01-school-head.md](../agents/01-school-head.md)

**Primary responsibilities**
- Approves policy-sensitive actions (grade overrides, scholarship awards, report-card publication, invoice cancellations).
- Oversees cohort KPIs and cross-department trends.
- Signs off on external reporting.

**Daily workflow**
1. Open the leadership dashboard — review attendance %, fee-collection %, flagged grades, at-risk students.
2. Process the approval queue (grades, scholarships, report cards, cancellations).
3. Check safeguarding alerts routed from `school_psychologist` and `hostel_parent`.
4. Read and respond to escalated parent communications.

**Escalation paths**
- Safeguarding incidents → `school_psychologist`.
- Compliance issues → `security_officer` + external counsel.
- Financial anomalies → `finance_bursar`.

---

## Operations

### registrar

Agent: [.claude/agents/02-registrar.md](../agents/02-registrar.md)

**Primary responsibilities**
- Student admissions, transfers, withdrawals, graduation.
- Maintains the canonical student record.
- Generates transcripts and term rollovers.

**Daily workflow**
1. Process new admissions: create Student, link Guardian, enrol in Class(es).
2. Handle transfers: close Enrolment, issue transcript package.
3. Update guardian contact changes; verify identity.
4. Reconcile class rosters with teachers before term start.

**End-of-term**
- Drive the term rollover: close Term, promote cohorts per BR-PRM-001, create the next Term's Classes.

---

### examinations_officer

Agent: [.claude/agents/02-examinations-officer.md](../agents/02-examinations-officer.md)

**Primary responsibilities**
- Schedules exams, invigilation, marking, moderation.
- Approves Grade state transitions.
- Guardian of exam integrity.

**Daily workflow**
1. Build the exam schedule; publish to teachers and students.
2. Assign invigilators; resolve clashes.
3. Moderate submitted grades (BR-GRD-002): approve, return for revision, or flag conflict.
4. Publish results under the approval flow once `school_head` signs off on report cards.

---

### finance_bursar

Agent: [.claude/agents/02-finance-bursar.md](../agents/02-finance-bursar.md)

**Primary responsibilities**
- Fee heads, fee schedules, invoicing, payments, receipts, reconciliation.
- ZIMRA fiscalisation compliance.

**Daily workflow**
1. Raise new Invoices at term start — one per Student × Term.
2. Record Payments (cash, EcoCash, OneMoney, bank, Paynow) — always with `Idempotency-Key` on digital channels.
3. Confirm pending mobile-money Payments via the Paynow callback or explicit reconciliation.
4. Issue Receipts and submit to the fiscal device (BR-RCP-002).
5. Propose Scholarships for leadership approval (BR-SCH-003).

**Monthly**
- Open and close the Reconciliation period (BR-REC-001/002). Resolve conflict-flagged Payments before closing.

---

## Teaching & Support

### teacher

Agent: [.claude/agents/03-teacher.md](../agents/03-teacher.md)

**Primary responsibilities**
- Lessons, attendance, assignments, grading, communication with students and guardians.

**Daily workflow**
1. Mark attendance at lesson start (web or mobile — offline-safe).
2. Deliver the lesson; log any adjustments.
3. Create or open Assessments; collect Submissions.
4. Grade submissions; draft enters `draft`, submit for `examinations_officer` approval.
5. Review AI-assisted grade suggestions; counter-sign before submitting (BR-AI-004).
6. Message students/guardians through the notifications channel.

---

### lms_specialist

Agent: [.claude/agents/03-learning-management-systems-specialist.md](../agents/03-learning-management-systems-specialist.md)

**Primary responsibilities**
- Course catalogue, lesson authoring, question banks, content versioning.
- Curriculum alignment and reusable assessment components.

**Weekly workflow**
1. Author or revise Lessons; move draft → published when ready.
2. Maintain QuestionBanks; retire low-quality items based on analytics.
3. Support teachers wiring assessments to the bank.

---

### hostel_parent

Agent: [.claude/agents/03-hostel-parent.md](../agents/03-hostel-parent.md)

**Primary responsibilities**
- Boarder welfare, discipline, daily routines; bridges academic and home concerns.

**Daily workflow**
1. Morning and evening roll — confirm boarders present.
2. Log welfare notes and incidents; escalate safeguarding concerns to `school_psychologist` or `school_head`.
3. Communicate with guardians about boarding-specific matters.

---

### school_psychologist

Agent: [.claude/agents/03-school-psychological-officer.md](../agents/03-school-psychological-officer.md)

**Primary responsibilities**
- Student mental health, safeguarding, behavioural/emotional wellbeing.
- Confidential notes; strict ethical access.

**Workflow**
1. Manage referrals from teachers, `hostel_parent`, or guardians.
2. Keep session notes in the confidential store (see BR-COM-002).
3. Raise safeguarding escalations to `school_head` and, where required, external authorities.
4. Coordinate interventions with teachers without breaching confidentiality.

---

## Users

### student

Agent: [.claude/agents/04-student.md](../agents/04-student.md)

**Primary workflows**
1. View timetable and lesson content (offline-capable on mobile).
2. Submit Assignments; take Quizzes (offline-capable; syncs on reconnect).
3. Check grades when released and Report Cards when published.
4. View Invoices (read-only) and push payment requests to Guardians.
5. Read announcements and messages.

---

### parent_guardian

Agent: [.claude/agents/04-parent-guardian.md](../agents/04-parent-guardian.md)

**Primary workflows**
1. View each linked Student's attendance, grades (when approved), report cards (when published), and fee balance.
2. Pay Invoices via Paynow / EcoCash / OneMoney / bank transfer.
3. Communicate with teachers and school; receive leadership announcements.
4. Acknowledge required permissions (field trips, policy updates).

---

## Quality & Technology

### qa_engineer

Agent: [.claude/agents/05-qa-test-engineer.md](../agents/05-qa-test-engineer.md)

**Primary responsibilities**
- Test strategy, edge cases, regression coverage.
- Challenges assumptions; verifies under adversarial conditions.

**Ongoing workflow**
1. Review every feature PR for test adequacy (see [09-TestingGuide.md](09-TestingGuide.md)).
2. Maintain the regression test suite.
3. Drive load scenarios and DR drills.
4. Coordinate tenant-isolation regression before each release.

---

### security_officer

Agent: [.claude/agents/05-security-privacy-officer.md](../agents/05-security-privacy-officer.md)

**Primary responsibilities**
- Access control, audit review, data protection, incident response.
- Regulatory compliance posture (DPA, ZIMRA, Ministry reporting).

**Ongoing workflow**
1. Review audit logs for anomalies; run periodic sampling.
2. Rotate secrets per [08-SecurityGuide.md §5](08-SecurityGuide.md).
3. Coordinate incident response (§10 of the Security Guide).
4. Approve data exports and right-to-erasure requests.

---

### system_designer

Agent: [.claude/agents/05-system-designer-developer.md](../agents/05-system-designer-developer.md)

**Primary responsibilities**
- Translates business requirements into modules, entities, APIs, workflows, and implementation plans.
- Owns architectural coherence and produces ADRs.

**Ongoing workflow**
1. Turn approved scope into ADRs, API contracts, and implementation tasks.
2. Guide or implement code aligned with [01-Architecture.md](01-Architecture.md).
3. Keep [13-ProjectStatus.md](13-ProjectStatus.md) and [11-Roadmap.md](11-Roadmap.md) current.
4. Review cross-module impact on any change.

---

## Common Escalation Ladder

```
incident observed
     │
     ├── safeguarding ──────────▶ school_psychologist ─▶ school_head ─▶ external authority
     ├── compliance / privacy ─▶ security_officer ────▶ school_head
     ├── academic integrity ───▶ examinations_officer ▶ school_head
     ├── financial anomaly ────▶ finance_bursar ──────▶ school_head
     └── technical defect ─────▶ system_designer ────▶ qa_engineer
```

---

## Status

Task flows above are the intended procedures. Client UIs (web and mobile) that realise these flows are **Not yet implemented**. Update each section with screenshots and click-by-click detail as each module ships.
