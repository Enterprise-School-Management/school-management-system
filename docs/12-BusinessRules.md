# 12 — Business Rules

Operational, academic, and financial rules that the School Management System must enforce. Use this as the source of truth when implementing validations, state transitions, and approval workflows.

**See also:** [06-DomainModel.md](06-DomainModel.md) · [08-SecurityGuide.md](08-SecurityGuide.md) · [03-ApiReference.md](03-ApiReference.md)

---

## Rule Numbering

Each rule has a stable code `BR-<DOMAIN>-<NNN>` so it can be referenced from code comments, tests, and ADRs. When a rule changes, bump its version in [02-ChangeLog.md](02-ChangeLog.md) and keep the code.

---

## 1. Cross-Cutting Rules

| Code | Rule | Source |
|------|------|--------|
| BR-CORE-001 | Every write operation produces one `AuditLog` entry in the same DB transaction. | Audit requirement |
| BR-CORE-002 | All tenant-scoped reads and writes are filtered by the authenticated user's active `school_id`. No endpoint may accept `school_id` as a parameter. | Multi-tenancy |
| BR-CORE-003 | AI output is advisory. No automated action (grade posting, discipline decision, payment release) may occur without a human actor. | AI-advisory principle |
| BR-CORE-004 | Deletions are soft (`deleted_at`) unless executing a lawful right-to-erasure request. | Audit / DPA |
| BR-CORE-005 | Grade and payment records use conflict flags rather than last-write-wins on sync conflicts. | Offline sync |
| BR-CORE-006 | Rows that pass a foreign key to a soft-deleted parent are not returned by default queries. | Data integrity |

---

## 2. Operational Rules

### Enrolment

| Code | Rule |
|------|------|
| BR-ENR-001 | A Student may have at most one `active` Enrolment in the same Class at a time. |
| BR-ENR-002 | Enrolment in a Class requires the Class's Term to be current or future (no retro-enrolment). |
| BR-ENR-003 | Withdrawing a Student from a Class marks all their `scheduled` attendance future-dated as `excused`. |
| BR-ENR-004 | Transferring a Student between schools preserves the original academic record read-only; the receiving school creates a fresh Enrolment chain. |
| BR-ENR-005 | A Student cannot be graduated before the terminal Term of their cohort's year. |

### Attendance

| Code | Rule |
|------|------|
| BR-ATT-001 | Attendance is recorded per Class per calendar date at most once per Student. |
| BR-ATT-002 | Attendance can be marked up to 7 days after the date; edits older than 7 days require `school_head` approval with reason. |
| BR-ATT-003 | A mark of `late` auto-converts to `present` for totalling purposes; the `late` flag is retained separately for reporting. |
| BR-ATT-004 | Mobile-submitted attendance carries `sync_source = mobile` and the device timestamp; the server records its own receipt time separately. |

### Timetabling

| Code | Rule |
|------|------|
| BR-TT-001 | A Teacher cannot be assigned to two Classes whose TimetableSlots overlap. |
| BR-TT-002 | A Class cannot have overlapping TimetableSlots in the same Room. |

### Communications

| Code | Rule |
|------|------|
| BR-COM-001 | Notifications sent to minors route through their Guardian unless the role sending is `school_psychologist` acting on a safeguarding matter. |
| BR-COM-002 | Counsellor ↔ student messages are confidential by default; visible only to the counsellor and, with explicit reason, to `school_head`. |

---

## 3. Academic Rules

### Grading

| Code | Rule |
|------|------|
| BR-GRD-001 | A Grade is created in `draft` by the teacher. Publication to the gradebook requires a transition to `submitted` then `approved`. |
| BR-GRD-002 | Only `examinations_officer` or `school_head` may approve Grades. |
| BR-GRD-003 | Editing an `approved` Grade requires `school_head` approval with a reason; the old value is retained in audit. |
| BR-GRD-004 | Conflict-flagged Grades are not counted toward the gradebook until resolved. |
| BR-GRD-005 | Letter-grade bands are configurable per school and versioned per academic year. |

### Report Cards

| Code | Rule |
|------|------|
| BR-RPT-001 | A Report Card may be generated only after all Grades for the Term are `approved`. |
| BR-RPT-002 | Publication requires `school_head` approval. |
| BR-RPT-003 | A published Report Card is immutable; corrections are issued as a versioned reissue, with the prior version retained. |

### Promotion

| Code | Rule |
|------|------|
| BR-PRM-001 | A Student is promoted if they meet the school's promotion criteria for the terminal Term of the year. Criteria are stored per school. |
| BR-PRM-002 | Promotion decisions are approved by `school_head`; overrides are audited with reason. |

### Assessment

| Code | Rule |
|------|------|
| BR-ASM-001 | Submissions accepted only while Assessment status is `open`. |
| BR-ASM-002 | Late submissions (post-`closed`) require a reason and `teacher` acknowledgement. |
| BR-ASM-003 | Exam-type Assessments require `examinations_officer` approval to transition to `closed`. |

---

## 4. Financial Rules

### Invoicing

| Code | Rule |
|------|------|
| BR-INV-001 | An Invoice is issued against exactly one Student for exactly one Term. |
| BR-INV-002 | Invoice totals equal the sum of InvoiceLines. A Payment never alters the total. |
| BR-INV-003 | Invoice state transitions follow the state machine in [06-DomainModel.md](06-DomainModel.md). `paid` is terminal except for authorised cancellation. |
| BR-INV-004 | An Invoice becomes `overdue` at `due_at` if `balance > 0`. A Celery job sweeps nightly. |
| BR-INV-005 | Cancelling an Invoice requires `finance_bursar` submit and `school_head` approval when `balance != total` (i.e. payment already received). |

### Payments

| Code | Rule |
|------|------|
| BR-PAY-001 | Every Payment records a non-empty `reference` (receipt, transaction ID, or manual note). |
| BR-PAY-002 | `cash` Payments require physical receipt number entry. |
| BR-PAY-003 | `paynow`, `ecocash`, and `onemoney` Payments are confirmed via callback or explicit reconciliation — they remain `pending` until then. |
| BR-PAY-004 | A Payment cannot exceed the Invoice balance at the time of submission. Overpayments are recorded as a separate `credit` entry (**Not yet implemented**; design owned by `finance_bursar`). |
| BR-PAY-005 | Refunds are new Payment rows with negative amounts; they require `school_head` approval. |

### Receipts & ZIMRA Fiscalisation

| Code | Rule |
|------|------|
| BR-RCP-001 | Every confirmed Payment produces exactly one Receipt. |
| BR-RCP-002 | Receipts for `tuition` and other ZIMRA-classified fee categories are submitted to the fiscal device and carry a `fiscal_reference`. |
| BR-RCP-003 | Fiscalisation failures queue the Receipt for retry; the Payment is not blocked. The unfiscalised Receipt is flagged in reconciliation. |
| BR-RCP-004 | Receipts are immutable once issued. Corrections are issued as a reversal + new Receipt, both linked. |

### Scholarships

| Code | Rule |
|------|------|
| BR-SCH-001 | A Scholarship reduces one or more Fee Heads by a percent or fixed amount per Term. |
| BR-SCH-002 | A Scholarship is applied automatically when generating Invoices for the affected Term. |
| BR-SCH-003 | Scholarship approval is by `school_head`; proposal may come from `finance_bursar` or `registrar`. |

### Reconciliation

| Code | Rule |
|------|------|
| BR-REC-001 | Reconciliation windows are monthly. A window cannot be closed while conflict-flagged Payments exist. |
| BR-REC-002 | Closing a reconciliation locks the period: no new Payments or Invoice edits dated inside the period without `school_head` approval. |

---

## 5. Data Sync Rules

| Code | Rule |
|------|------|
| BR-SYN-001 | Mobile `push` requests are idempotent: replays with the same `client_operation_id` are no-ops. |
| BR-SYN-002 | Server-side sync reconciler applies last-write-wins for low-risk data and conflict-flag for Grade and Payment. |
| BR-SYN-003 | A device may only sync as the user currently authenticated on it; token switch clears the local sync queue after successful upload. |
| BR-SYN-004 | Sync conflicts on Grades and Payments are resolved via `POST /sync/resolve` by an authorised role (see [08-SecurityGuide.md](08-SecurityGuide.md)). |

---

## 6. Reporting Obligations

| Code | Rule |
|------|------|
| BR-RPT-EXT-001 | Ministry of Primary and Secondary Education reports are generated on a schedule; exact cadence **TBD** pending school confirmation. |
| BR-RPT-EXT-002 | ZIMRA fiscal daily totals are reported per fiscal-device policy. |
| BR-RPT-EXT-003 | Leadership dashboards publish cohort KPIs (pass rates, attendance %, fee-collection %) without exposing individual PII in cross-cohort views. |

---

## 7. AI-Advisory Rules

| Code | Rule |
|------|------|
| BR-AI-001 | AI recommendations are recorded as `Recommendation` rows with a TTL; they never execute actions directly. |
| BR-AI-002 | Risk flags are visible only to roles with legitimate access to the target student (see scope modifiers in [08-SecurityGuide.md](08-SecurityGuide.md)). |
| BR-AI-003 | Feedback drafts from NLP models are presented as suggestions in the teacher UI; they are not auto-applied. |
| BR-AI-004 | Any AI-assisted grade must be counter-signed by the teacher before it enters `submitted`. |

---

## 8. Status

Rules above are the current design baseline as of 2026-04-17. They will be revised as stakeholders review [11-Roadmap.md](11-Roadmap.md) phases; confirmed revisions are logged in [02-ChangeLog.md](02-ChangeLog.md) with the affected `BR-` codes.
