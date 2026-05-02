# 08 — Security Guide

Authentication, authorisation, audit, secrets, and compliance baseline for the School Management System.

**See also:** [01-Architecture.md](01-Architecture.md) · [07-DatabaseGuide.md](07-DatabaseGuide.md) · [12-BusinessRules.md](12-BusinessRules.md) · [10-DeploymentGuide.md](10-DeploymentGuide.md)

---

## 1. Authentication

- **Primary:** Django Auth + JWT. No cloud SSO dependency for local deployment.
- **Token types:**
  - Access token — short-lived (15 min), bearer in `Authorization` header.
  - Refresh token — longer-lived (7 days), stored HttpOnly on web, Keychain/Keystore on mobile.
- **Claims:** `user_id`, `school_id` (active membership), `role`, `issued_at`, `exp`, `jti`.
- **Token rotation:** refresh returns a new access + new refresh; old refresh is revoked.
- **Password policy:** minimum 10 characters, must not match common-password list, rotated annually for staff.
- **MFA:** TOTP for `school_head`, `finance_bursar`, `security_officer`, `system_designer`. **Not yet implemented** — scheduled Phase 3.

---

## 2. Roles & Permission Matrix

Thirteen roles map to the agent definitions under [.claude/agents/](../agents/).

| Role | Users | Courses | Enrolments | Assessments | Billing | Analytics | Notifications | Audit | Superadmin |
|------|-------|---------|------------|-------------|---------|-----------|---------------|-------|------------|
| **school_head** | R | R | R | R + Approve | R | R | R + Send | R | — |
| **registrar** | CRUD | R | CRUD | R | R | R | R + Send | R | — |
| **examinations_officer** | R | R | R | CRUD + Approve | — | R | R + Send | R | — |
| **finance_bursar** | R | — | R | — | CRUD + Approve | R | R + Send | R | — |
| **teacher** | R (own) | R | R (own classes) | CRUD (own) | — | R (own) | R + Send (own) | — | — |
| **lms_specialist** | R | CRUD | R | R | — | R | R | — | — |
| **hostel_parent** | R (boarders) | — | R (boarders) | — | R (boarders) | R | R + Send (boarders) | — | — |
| **school_psychologist** | R (referred) | — | R (referred) | R (referred) | — | R (referred) | R + Send (confidential) | — | — |
| **qa_engineer** | R | R | R | R | R | R | R | R | — |
| **security_officer** | R | R | R | R | R | R | R | R | R |
| **system_designer** | CRUD | CRUD | CRUD | CRUD | CRUD | CRUD | CRUD | R | R |
| **student** | R (self) | R (enrolled) | R (self) | R (own) + Submit | R (own) | R (self) | R (self) | — | — |
| **parent_guardian** | R (self, linked) | — | R (linked) | R (linked) | R (linked) + Pay | R (linked) | R (self) | — | — |

**Legend:** `R` read · `C` create · `U` update · `D` delete (soft) · `—` no access · `Approve` state-transition approval · `Send` send notifications · `Pay` make payments.

Scope modifiers: `(own)`, `(self)`, `(linked)`, `(boarders)`, `(referred)`, `(enrolled)` — the role sees only the rows explicitly tied to them.

> **Authoritative list:** this matrix is the single source of permission truth. When adding an endpoint, update this table and the endpoint's permission decorator together. Discrepancies are bugs.

---

## 3. Scoping Layers

Defence in depth — a request must pass all three:

1. **Authentication** — valid JWT.
2. **Role permission** — endpoint decorator checks `RoleCode` against the matrix above.
3. **Tenant scope** — ORM manager filters `school_id` against the authenticated user's active membership; see [07-DatabaseGuide.md §6](07-DatabaseGuide.md).
4. **Row scope** — for `(own)` / `(self)` / `(linked)` modifiers, service checks row ownership.

---

## 4. Audit Logging

- Every write operation emits an `AuditLog` row in the same transaction as the write.
- Fields: `school_id`, `actor_user_id`, `action`, `target_type`, `target_id`, `before` (jsonb), `after` (jsonb), `ip_address`, `user_agent`, `at`.
- Append-only. Never updated, never soft-deleted.
- **Elevated actions** require additional fields: `reason_text` (required), `approved_by` (required on approval flows). Applies to: grade override, grade approval, report-card publication, scholarship approval, invoice cancellation, fee-schedule change, user role change, data export.
- Audit search endpoint `/audit` is gated to `security_officer` and `system_designer` within a school, `superadmin` across schools.

---

## 5. Secrets Management

| Secret | Where It Lives | Rotation |
|--------|----------------|----------|
| Django `SECRET_KEY` | `.env` on school server; generated per install | On any suspected leak |
| DB password | `.env`; never in code | Quarterly |
| JWT signing key | `.env`; distinct per install | Annually or on incident |
| Paynow integration key | `.env`; per-school | On rotation request |
| SMS gateway credentials | `.env`; per-school | Per provider policy |
| Cloud-backup credentials | `.env`; encrypted-at-rest | Annually |

Rules:
- **Never commit `.env`.** The repo ships `.env.example` only.
- Secrets in CI live in GitHub Actions encrypted secrets.
- Log redaction middleware strips secret-like strings before any log write.
- Rotate immediately on suspected compromise; document the event in [02-ChangeLog.md](02-ChangeLog.md) under "Security".

---

## 6. Transport Security

- **School LAN:** TLS required — self-signed CA distributed to school devices during provisioning.
- **Certificates:** rotated annually; automation via school-server setup script (**Not yet implemented**).
- **Mobile pinning:** client pins the school-server certificate on first connect; manual re-trust required on rotation.
- **Internet-bound connections:** only for cloud backup, SMS, Paynow, email — all TLS-1.2+ with verification.

---

## 7. Data Protection

### Personally Identifiable Information (PII)

- Encrypted at rest at the filesystem level (LUKS on Linux school server).
- Student dates of birth, addresses, guardian phone numbers treated as PII.
- Counsellor notes (from `school_psychologist`) treated as **sensitive PII** — access restricted to that role and `school_head` with explicit reason.

### Data retention

| Class of data | Retention |
|---------------|-----------|
| Student academic records | Indefinite (legal obligation) |
| Attendance records | 7 years |
| Financial records | 7 years (ZIMRA requirement) |
| Audit logs | 7 years |
| Counselling notes | Until student leaves + 2 years, unless safeguarding exception |
| Session / access logs | 90 days |

### Right to erasure

- Requested via admin flow; requires `school_head` approval and `security_officer` execution.
- Scrubs PII fields; retains aggregate academic/financial rows with pseudonymised identifier.
- Audit entry records the erasure.

---

## 8. Compliance

| Framework | Applies | Notes |
|-----------|---------|-------|
| **Zimbabwe Cyber Security and Data Protection Act (DPA)** | Always | Governs PII handling, breach notification, data-subject rights. |
| **ZIMRA Fiscalisation** | Billing module | Certain receipts must pass through a fiscal device. See [12-BusinessRules.md](12-BusinessRules.md). |
| **Ministry of Primary and Secondary Education reporting** | Always | Required reports; defined in [12-BusinessRules.md](12-BusinessRules.md). |
| **GDPR / FERPA / COPPA** | Only on international expansion | Not enforced until/unless deployed outside Zimbabwe. |

---

## 9. CI Security Gates

Every deployment pipeline must pass:

1. **Lint** — Ruff + Black.
2. **SAST** — Bandit for Python; Semgrep rules for common web flaws.
3. **Dependency vulnerability scan** — `pip-audit` / `npm audit` at CI.
4. **Container image scan** — Trivy on the built image.
5. **Secrets scan** — gitleaks on every push.
6. **Human review** — at least one reviewer for every AI-generated change.

A failure on any gate blocks merge.

---

## 10. Incident Response

1. **Detect** — alerts from audit logs, failed-auth spikes, or user report.
2. **Contain** — rotate affected secrets; revoke tokens; disable user if necessary.
3. **Investigate** — pull audit trail; preserve logs.
4. **Notify** — DPA breach notification where triggered; `school_head` always; affected users per severity.
5. **Recover** — restore from backup if data integrity is affected; see [10-DeploymentGuide.md](10-DeploymentGuide.md).
6. **Document** — post-incident note in [02-ChangeLog.md](02-ChangeLog.md) under "Security".

---

## 11. Safeguarding

- Counsellor and welfare notes (`school_psychologist`, `hostel_parent`) are flagged as confidential.
- Any child-safeguarding concern triggers a mandatory reporting workflow — **Not yet implemented**; design owned by the [school_psychological_officer](../agents/03-school-psychological-officer.md) agent.
- Audit entries for safeguarding actions are segregated from the general audit stream.

---

## 12. Status

All controls above are design-stage unless marked otherwise. Implementation is staged in [11-Roadmap.md](11-Roadmap.md), with core RBAC and audit logging landing in Phase 1.
