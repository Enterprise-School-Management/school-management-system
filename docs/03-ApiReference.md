# 03 — API Reference

REST endpoint reference for every domain module. Conventions first, then endpoints grouped by module.

**See also:** [01-Architecture.md](01-Architecture.md) · [06-DomainModel.md](06-DomainModel.md) · [08-SecurityGuide.md](08-SecurityGuide.md)

> **Status legend:** `Planned` — in design. `Drafted` — OpenAPI spec exists. `Implemented` — code exists in the backend. All endpoints below are **Planned** unless noted otherwise.

---

## 1. Conventions

### Base path

```
/api/v1/
```

All endpoints are versioned. Breaking changes bump to `v2`; non-breaking additions stay on `v1`.

### Authentication

- Bearer JWT in `Authorization: Bearer <token>` header.
- Tokens issued at `POST /api/v1/auth/login`.
- Tenant (`school_id`) is embedded in the token claims — clients never pass it as a parameter.

### Content type

- `application/json` for requests and responses.
- `multipart/form-data` for file uploads (attachments).

### Standard response shape

```json
// success
{ "data": { ... }, "meta": { ... } }

// list
{ "data": [ ... ], "meta": { "page": 1, "page_size": 25, "total": 431 } }

// error
{ "error": { "code": "validation_error", "message": "…", "details": [ … ] } }
```

### Pagination

- Cursor or offset; default `page=1`, `page_size=25`, max `page_size=100`.
- `meta.next_cursor` / `meta.prev_cursor` returned when cursor pagination is used.

### Filtering & sorting

- Query string: `?status=active&grade_level=form_2&sort=-created_at`.
- Leading `-` in `sort` reverses the order.

### Error codes

| Code | HTTP | Meaning |
|------|------|---------|
| `validation_error` | 400 | Request failed field validation. `details[]` lists offenders. |
| `unauthenticated` | 401 | No or invalid token. |
| `forbidden` | 403 | Authenticated but role lacks permission. |
| `not_found` | 404 | Resource absent or not in this tenant. |
| `conflict` | 409 | State transition not allowed (see state machines in [06-DomainModel.md](06-DomainModel.md)). |
| `rate_limited` | 429 | Client exceeded quota. |
| `server_error` | 500 | Unexpected — logged and audited. |

### Idempotency

- Every `POST` that creates a resource accepts `Idempotency-Key` header (required for payments, optional elsewhere).

### Sync endpoints

- Mobile sync has its own root at `/api/v1/sync/` with different pagination and conflict semantics; see §10.

---

## 2. Auth

| Method | Path | Purpose | Status |
|--------|------|---------|--------|
| POST | `/auth/login` | Exchange credentials for JWT. | Planned |
| POST | `/auth/refresh` | Refresh an access token. | Planned |
| POST | `/auth/logout` | Revoke refresh token. | Planned |
| POST | `/auth/password-reset` | Request password-reset email. | Planned |
| POST | `/auth/password-reset/confirm` | Complete password reset. | Planned |

---

## 3. Users

| Method | Path | Purpose | Status |
|--------|------|---------|--------|
| GET | `/users/me` | Current user + role + active school. | Planned |
| GET | `/users` | List users in current school. | Planned |
| POST | `/users` | Create user + SchoolMembership. | Planned |
| GET | `/users/{id}` | Fetch user. | Planned |
| PATCH | `/users/{id}` | Update profile or role. | Planned |
| DELETE | `/users/{id}` | Soft-delete user. | Planned |
| POST | `/users/{id}/memberships` | Add user to another school (superadmin). | Planned |
| GET | `/schools` | List schools the current user belongs to. | Planned |
| GET | `/schools/{id}` | Fetch school metadata and active modules. | Planned |

---

## 4. Courses

| Method | Path | Purpose | Status |
|--------|------|---------|--------|
| GET | `/courses` | List courses. | Planned |
| POST | `/courses` | Create course. | Planned |
| GET | `/courses/{id}` | Fetch course. | Planned |
| PATCH | `/courses/{id}` | Update course. | Planned |
| GET | `/courses/{id}/lessons` | List lessons. | Planned |
| POST | `/courses/{id}/lessons` | Create lesson. | Planned |
| PATCH | `/lessons/{id}` | Update lesson. | Planned |
| POST | `/lessons/{id}/publish` | Transition draft → published. | Planned |
| GET | `/question-banks` | List banks. | Planned |
| POST | `/question-banks` | Create bank. | Planned |
| GET | `/question-banks/{id}/questions` | List questions. | Planned |
| POST | `/question-banks/{id}/questions` | Add question. | Planned |

---

## 5. Enrolments

| Method | Path | Purpose | Status |
|--------|------|---------|--------|
| GET | `/students` | List students. | Planned |
| POST | `/students` | Register student. | Planned |
| GET | `/students/{id}` | Fetch student. | Planned |
| PATCH | `/students/{id}` | Update student. | Planned |
| GET | `/students/{id}/guardians` | List guardians. | Planned |
| POST | `/students/{id}/guardians` | Link guardian. | Planned |
| GET | `/academic-years` | List academic years. | Planned |
| GET | `/terms` | List terms. | Planned |
| GET | `/classes` | List classes. | Planned |
| POST | `/classes` | Create class. | Planned |
| GET | `/classes/{id}` | Fetch class. | Planned |
| GET | `/classes/{id}/roster` | Students enrolled in class. | Planned |
| POST | `/classes/{id}/enrolments` | Enrol student. | Planned |
| PATCH | `/enrolments/{id}` | Update enrolment status. | Planned |
| GET | `/classes/{id}/attendance` | Attendance roster for a date. | Planned |
| POST | `/classes/{id}/attendance` | Submit attendance (offline-safe). | Planned |
| GET | `/classes/{id}/timetable` | Fetch weekly timetable. | Planned |

---

## 6. Assessments

| Method | Path | Purpose | Status |
|--------|------|---------|--------|
| GET | `/assessments` | List assessments. | Planned |
| POST | `/assessments` | Create assessment. | Planned |
| GET | `/assessments/{id}` | Fetch assessment. | Planned |
| PATCH | `/assessments/{id}` | Update or transition status. | Planned |
| POST | `/assessments/{id}/open` | Open for submissions. | Planned |
| POST | `/assessments/{id}/close` | Close submissions. | Planned |
| GET | `/assessments/{id}/submissions` | List submissions. | Planned |
| POST | `/assessments/{id}/submissions` | Submit (offline-safe). | Planned |
| GET | `/submissions/{id}` | Fetch submission. | Planned |
| POST | `/submissions/{id}/grade` | Grade a submission. | Planned |
| GET | `/grades` | List grades. | Planned |
| PATCH | `/grades/{id}` | Update grade (generates audit entry). | Planned |
| POST | `/grades/{id}/approve` | Approve (leadership). | Planned |
| POST | `/grades/{id}/flag-conflict` | Raise conflict flag. | Planned |
| GET | `/classes/{id}/gradebook` | Class gradebook view. | Planned |
| GET | `/enrolments/{id}/report-card` | Report card for a term. | Planned |
| POST | `/report-cards/{id}/publish` | Publish (leadership approval). | Planned |

---

## 7. Billing

| Method | Path | Purpose | Status |
|--------|------|---------|--------|
| GET | `/fee-heads` | List fee heads. | Planned |
| POST | `/fee-heads` | Create fee head. | Planned |
| GET | `/fee-schedules` | Current term schedule. | Planned |
| POST | `/fee-schedules` | Set term amount. | Planned |
| GET | `/invoices` | List invoices. | Planned |
| POST | `/invoices` | Raise invoice. | Planned |
| GET | `/invoices/{id}` | Fetch invoice + lines + payments. | Planned |
| POST | `/invoices/{id}/issue` | Transition draft → issued. | Planned |
| POST | `/invoices/{id}/cancel` | Cancel invoice. | Planned |
| GET | `/invoices/{id}/payments` | List payments. | Planned |
| POST | `/invoices/{id}/payments` | Record payment (requires `Idempotency-Key`). | Planned |
| GET | `/payments/{id}` | Fetch payment. | Planned |
| POST | `/payments/{id}/confirm` | Confirm a pending payment. | Planned |
| GET | `/receipts/{id}` | Fetch receipt. | Planned |
| POST | `/receipts/{id}/fiscalise` | Submit to ZIMRA fiscal device. | Planned |
| GET | `/scholarships` | List scholarships. | Planned |
| POST | `/scholarships` | Propose scholarship. | Planned |
| POST | `/scholarships/{id}/approve` | Approve. | Planned |
| POST | `/paynow/callback` | Webhook from Paynow. | Planned |
| POST | `/reconciliations` | Open reconciliation period. | Planned |
| POST | `/reconciliations/{id}/close` | Close reconciliation. | Planned |

---

## 8. Analytics

| Method | Path | Purpose | Status |
|--------|------|---------|--------|
| GET | `/analytics/engagement` | Engagement scores (filterable). | Planned |
| GET | `/analytics/risk-flags` | At-risk students. | Planned |
| POST | `/analytics/risk-flags/{id}/acknowledge` | Acknowledge a flag. | Planned |
| GET | `/analytics/recommendations` | Current recommendations for the caller. | Planned |
| GET | `/analytics/dashboards/{kind}` | Dashboard payload. | Planned |

---

## 9. Notifications

| Method | Path | Purpose | Status |
|--------|------|---------|--------|
| GET | `/notifications` | Inbox for caller. | Planned |
| POST | `/notifications/{id}/read` | Mark as read. | Planned |
| POST | `/notifications/send` | Send ad-hoc notification (role-gated). | Planned |
| GET | `/notification-templates` | List templates. | Planned |
| POST | `/notification-templates` | Create template. | Planned |
| GET | `/notification-preferences` | Caller's preferences. | Planned |
| PATCH | `/notification-preferences` | Update preferences. | Planned |

---

## 10. Mobile Sync

| Method | Path | Purpose | Status |
|--------|------|---------|--------|
| POST | `/sync/pull` | Fetch server changes since cursor. | Planned |
| POST | `/sync/push` | Upload device mutations. | Planned |
| POST | `/sync/resolve` | Resolve a conflict-flagged record. | Planned |

Sync semantics: see [01-Architecture.md §7](01-Architecture.md). Grades and payments always return conflict-flag results rather than overwriting.

---

## 11. Superadmin (cross-school)

| Method | Path | Purpose | Status |
|--------|------|---------|--------|
| GET | `/super/schools` | All schools on this installation. | Planned |
| GET | `/super/metrics` | Cross-school aggregates. | Planned |
| GET | `/super/audit` | Cross-school audit search. | Planned |

Superadmin endpoints bypass the tenant filter and require `role = system_designer` or `role = security_officer` with a superadmin flag. See [08-SecurityGuide.md](08-SecurityGuide.md).

---

## 12. Health & Ops

| Method | Path | Purpose | Status |
|--------|------|---------|--------|
| GET | `/health` | Liveness. | Planned |
| GET | `/health/ready` | Readiness (DB, Redis reachable). | Planned |
| GET | `/version` | Build metadata. | Planned |
