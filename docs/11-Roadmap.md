# 11 — Roadmap

Phased feature plan from foundation to production readiness. Derived from the three-month delivery plan in [docs/spec.md §12](../../docs/spec.md).

**See also:** [13-ProjectStatus.md](13-ProjectStatus.md) · [01-Architecture.md](01-Architecture.md) · [docs/spec.md](../../docs/spec.md)

---

## Phase Map

```
Phase 0 — Foundation          (Week 1–2)
Phase 1 — Core MVP            (Week 3–8)
Phase 2 — AI MVP              (Week 9–10)
Phase 3 — Hardening & Launch  (Week 11–12)
Phase 4+ — Deferred features  (post-launch)
```

Each phase has a **Goal**, **Deliverables**, and **Exit criteria**. A phase ends only when all exit criteria are met.

---

## Phase 0 — Foundation (Weeks 1–2)

**Goal:** Lock decisions, stand up infrastructure, make the next 10 weeks executable.

**Deliverables**
- Requirements review signed off by `school_head`, `finance_bursar`, `registrar`, `examinations_officer`.
- Domain model and wireframes approved.
- ADR template adopted; first batch of ADRs written (DB engine choice, tenant model, JWT-vs-session, sync strategy).
- Security baseline documented (this doc set — [08-SecurityGuide.md](08-SecurityGuide.md)).
- Master system prompt constructed for multi-agent development.
- Docker Compose dev stack running locally (postgres, redis, django, celery, nginx).
- CI pipeline green: lint, SAST, dependency scan, image build.
- Repository layout: `apps/`, `core/`, `sync/` skeletons.

**Exit criteria**
- `docker compose up` starts a working dev stack end-to-end.
- CI blocks merges on any failing gate.
- Documentation set in `.claude/docs/` cross-linked and internally consistent.

---

## Phase 1 — Core MVP (Weeks 3–8)

**Goal:** Ship the modules required to run a real school end-to-end for one term.

### 1a — Identity & Tenancy (Weeks 3–4)

**Deliverables**
- Users module: `School`, `User`, `SchoolMembership`, `Role`, `Guardian`.
- JWT auth with refresh rotation.
- RBAC permission decorators covering the matrix in [08-SecurityGuide.md §2](08-SecurityGuide.md).
- Tenant scoping middleware + ORM manager.
- Tenant-leak regression suite (must be green).
- Audit log plumbing wired into every write path.

### 1b — Academic Core (Weeks 5–6)

**Deliverables**
- Enrolments module: `Student`, `AcademicYear`, `Term`, `Class`, `Enrolment`, `TimetableSlot`, `AttendanceRecord`.
- Courses module: `Course`, `Lesson`, `LessonVersion`, `QuestionBank`, `Question`.
- File upload with local-disk storage and attachment records.
- Announcements surface (simple broadcast via Notifications).
- Offline attendance on mobile (initial sync protocol).

### 1c — Assessments & Finance (Weeks 7–8)

**Deliverables**
- Assessments module: `Assessment`, `AssessmentItem`, `Submission`, `Grade`, `Gradebook`, `ReportCard`.
- Grade state machine + approval flow (BR-GRD-*).
- Billing module: `FeeHead`, `FeeSchedule`, `Invoice`, `InvoiceLine`, `Payment`, `Receipt`, `Scholarship`, `Reconciliation`.
- Paynow integration + callback handler.
- Notifications: in-app, email, SMS via local gateway; push via FCM.
- Leadership, teacher, and bursar dashboards (MVP views).

**Exit criteria (end of Phase 1)**
- All MVP endpoints in [03-ApiReference.md](03-ApiReference.md) §3–§9 marked `Implemented`.
- Tenant-isolation suite, RBAC matrix tests, grade state-machine tests, invoice/payment tests all green.
- Seed demo school renders every dashboard without error.
- Mobile app (Android only at this point) can mark attendance, submit an assignment, and view a gradebook — online and offline.

---

## Phase 2 — AI MVP (Weeks 9–10)

**Goal:** Deliver Phase 1 AI features per [docs/spec.md §6.1](../../docs/spec.md) — advisory only.

**Deliverables**
- Engagement scoring job (Celery).
- At-risk flag generation (rule-based in this phase).
- Recommendation rules and visibility.
- Leadership and teacher dashboards for engagement / risk.
- `Recommendation` entity and `RiskFlag` entity wired into the Analytics module.
- Predictive ML (dropout risk classification) **deferred to Phase 4** to avoid dataset-cold-start issues.

**Exit criteria**
- Dashboard loads and paints a populated seed school in under 500 ms on low-end Android target hardware.
- No AI path can execute an action without a human actor (BR-AI-001).

---

## Phase 3 — Hardening & Launch (Weeks 11–12)

**Goal:** Make the system safe to run in production at a real school.

**Deliverables**
- Load testing against §5 scenarios of [09-TestingGuide.md](09-TestingGuide.md).
- Security audit — external or structured internal.
- Accessibility audit — WCAG 2.1 AA checks on web and mobile.
- DR drill executed; RTO/RPO targets met.
- Mobile polish: low-end Android tuning, bundle-size reductions, offline UX refinement.
- TLS + cert-rotation automation on the school server.
- First school deployment dry-run.

**Exit criteria**
- Load scenarios pass within target latencies.
- Security audit findings triaged to `blocker`, `fast-follow`, `backlog`; all `blocker` items closed.
- DR drill completes in < 2 hours RTO.
- Go / no-go signoff from `school_head`, `security_officer`, `system_designer`.

---

## Phase 4+ — Deferred Features (Post-launch)

Not in MVP. Scheduled once the MVP has run for at least one full term.

| Track | Features |
|-------|----------|
| **AI — Phase 2** | Dropout risk classifier, learning-path personalisation, NLP grading assistance, sentiment analysis. |
| **AI — Phase 3** | RAG tutor over school content; policy Q&A assistant. |
| **Proctoring & integrity** | Advanced exam proctoring. |
| **Adaptive learning** | Deep adaptive pathways. |
| **Library module** | Full library management. |
| **Marketplace** | Complex marketplace and billing workflows. |
| **Cloud operations** | Kubernetes, ELK, Prometheus, Grafana. |
| **Microservices extraction** | Video processing, full-text search, AI inference, notification delivery. |

Deferrals are reviewed at each term boundary.

---

## Tracking

- **Status of each phase** — [13-ProjectStatus.md](13-ProjectStatus.md).
- **What shipped when** — [02-ChangeLog.md](02-ChangeLog.md).
- **Business rule changes** — annotated with `BR-*` codes in the changelog.

---

## Risks to Flag Early

1. **Dataset cold-start for AI** — Phase 2 classifier training needs at least one term of data. Mitigation: rule-based in MVP.
2. **Paynow sandbox parity with production** — schedule integration testing against the sandbox in Phase 1c.
3. **ZIMRA fiscal device availability on school servers** — procurement owned by `finance_bursar`; deadline = start of Phase 1c.
4. **Mobile device variety** — low-end Android tuning pushed to Phase 3 risks late-stage rework; early smoke tests in Phase 1b mitigate.
5. **Offline sync conflict semantics** — correctness of grade/payment conflict flagging is high-consequence; allocate targeted QA in Phase 1c.
