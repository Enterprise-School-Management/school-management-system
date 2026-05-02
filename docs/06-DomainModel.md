# 06 — Domain Model

Entities, properties, enums, relationships, and state machines for every domain module. The canonical list of what the system knows about.

**See also:** [01-Architecture.md](01-Architecture.md) · [07-DatabaseGuide.md](07-DatabaseGuide.md) · [03-ApiReference.md](03-ApiReference.md) · [12-BusinessRules.md](12-BusinessRules.md)

---

## Conventions

- Every entity carries: `id` (UUID), `created_at`, `updated_at`, `deleted_at?`.
- Every tenant-scoped entity carries: `school_id` (FK → `schools.id`).
- Status fields are enums declared explicitly per module.
- Aggregate roots are marked **(root)** — writes to child entities go through the root.

---

## Module — Schools

The **foundational module**. Schools is the root of the multi-tenant model. Every other module's data is scoped to a school via `school_id`. Schools owns school lifecycle, configuration, module activation flags, and TenantContext resolution.

### Entities

| Entity | Key Properties | Notes |
|--------|----------------|-------|
| **School** *(root)* | `name`, `short_code`, `address`, `timezone`, `locale`, `contact_email`, `contact_phone`, `logo_ref?`, `subscription_tier`, `is_active`, `registered_at` | Top-level tenant. Does not carry `school_id`. `short_code` is a unique URL slug used for subdomain resolution. |
| **SchoolConfig** | `school_id`, `key`, `value` | Key-value store for per-school settings (e.g. grading scale, academic calendar, ZIMRA fiscal number). |
| **FeatureFlag** | `school_id`, `module_code`, `is_enabled` | Plugin activation per school. Owned by Schools; read by all other modules. |

> `FeatureFlag` is promoted from Cross-Cutting Entities to Schools — Schools is its natural owner because module activation is a school administration decision.

### Enums

- `SubscriptionTier`: `free`, `standard`, `premium`.
- `ModuleCode`: `schools`, `users`, `courses`, `enrolments`, `assessments`, `billing`, `analytics`, `notifications`. Matches the module set in [01-Architecture.md §5](01-Architecture.md).

### Relationships

- `School` 1..* `FeatureFlag` (one flag record per activated module)
- `School` 1..* `SchoolConfig`
- All other modules reference `School` via `school_id` FK; they never embed a `School` object.

### School Creation Policy

| Actor | Capability |
|-------|------------|
| **Superadmin** | Create, activate, deactivate, and delete any school. |
| **Self-registration** | A public `/schools/register` endpoint is available when the global config flag `ALLOW_SCHOOL_SELF_REGISTRATION` is `true` (default: `false`). A self-registered school starts in `is_active = false`; a superadmin must approve it before users can log in. |

### Events Emitted

| Event | Payload | Subscribers |
|-------|---------|-------------|
| `school.created` | `school_id`, `name`, `tier` | Users (seed roles), Notifications (welcome message) |
| `school.activated` | `school_id` | All modules (enable access) |
| `school.deactivated` | `school_id`, `reason` | All modules (suspend access), Notifications (alert admin) |
| `school.module_toggled` | `school_id`, `module_code`, `is_enabled` | Billing (reprice), Users (recompute permissions) |
| `school.subscription_changed` | `school_id`, `old_tier`, `new_tier` | Billing (adjust plan) |

---

## Module — Users

### Entities

| Entity | Key Properties | Notes |
|--------|----------------|-------|
| **User** *(root)* | `email`, `password_hash`, `full_name`, `phone`, `is_active`, `last_login_at` | Global identity. A user may belong to multiple schools via SchoolMembership. |
| **SchoolMembership** | `user_id`, `school_id`, `role`, `started_at`, `ended_at?` | Binds a User to a School with a Role. `school_id` is a FK → `schools.id`. |
| **Role** | `code`, `name`, `permissions[]` | Enum-driven (see enum below); permissions bound at issue time. |
| **Guardian** | `user_id`, `relationship`, `linked_students[]` | A User acting as parent/guardian of one or more Students. |

### Enums

- `RoleCode`: `school_head`, `registrar`, `examinations_officer`, `finance_bursar`, `teacher`, `lms_specialist`, `hostel_parent`, `school_psychologist`, `qa_engineer`, `security_officer`, `system_designer`, `student`, `parent_guardian`. Matches the 13 roles in [.claude/agents/](../agents/).

### Relationships

- `User` 1..* `SchoolMembership` *..1 `School` (School owned by the Schools module)
- `SchoolMembership` *..1 `Role`
- `Guardian` *..* `Student` (via a join table)

---

## Module — Courses

### Entities

| Entity | Key Properties | Notes |
|--------|----------------|-------|
| **Course** *(root)* | `school_id`, `code`, `title`, `grade_level`, `syllabus`, `is_active` | Reusable subject definition. |
| **Lesson** | `course_id`, `order`, `title`, `content_ref`, `status` | Ordered inside a Course. |
| **LessonVersion** | `lesson_id`, `version`, `content_body`, `published_at?` | Draft / published flow. |
| **QuestionBank** | `school_id`, `course_id?`, `name` | Reusable across Assessments. |
| **Question** | `bank_id`, `type`, `stem`, `choices`, `answer_key`, `difficulty` | Question type enum below. |

### Enums

- `LessonStatus`: `draft`, `published`, `archived`.
- `QuestionType`: `multiple_choice`, `true_false`, `short_answer`, `essay`.

---

## Module — Enrolments

### Entities

| Entity | Key Properties | Notes |
|--------|----------------|-------|
| **Student** *(root)* | `school_id`, `student_number`, `full_name`, `date_of_birth`, `gender`, `guardians[]` | One of the two primary aggregate roots (with Invoice). |
| **AcademicYear** | `school_id`, `label`, `start_date`, `end_date` | e.g. "2026". |
| **Term** | `academic_year_id`, `number`, `start_date`, `end_date` | 1..3 per year typically. |
| **Class** | `school_id`, `course_id`, `term_id`, `teacher_id`, `room`, `capacity` | Concrete instance of a Course for a Term. |
| **Enrolment** *(root)* | `student_id`, `class_id`, `status`, `enrolled_on`, `withdrawn_on?` | Links Student to Class. |
| **TimetableSlot** | `class_id`, `day_of_week`, `start_time`, `end_time` | Weekly pattern. |
| **AttendanceRecord** | `enrolment_id`, `date`, `status`, `marked_by`, `marked_at`, `sync_source` | Offline-first; `sync_source ∈ {web, mobile}`. |

### Enums

- `EnrolmentStatus`: `active`, `withdrawn`, `graduated`, `transferred`.
- `AttendanceStatus`: `present`, `absent`, `late`, `excused`.

### State Machine — Enrolment

```
created → active ──────────┬─▶ withdrawn
                           ├─▶ graduated  (end of terminal year)
                           └─▶ transferred
```

---

## Module — Assessments

### Entities

| Entity | Key Properties | Notes |
|--------|----------------|-------|
| **Assessment** *(root)* | `class_id`, `type`, `title`, `weight`, `due_at`, `status` | Root for submissions + grades. |
| **AssessmentItem** | `assessment_id`, `question_id`, `order`, `points` | For structured quizzes/exams. |
| **Submission** | `assessment_id`, `student_id`, `submitted_at`, `content_ref`, `sync_source` | One per student per assessment. |
| **Grade** *(root)* | `submission_id?`, `enrolment_id`, `assessment_id`, `raw_score`, `letter_grade`, `comment`, `status`, `graded_by`, `approved_by?` | Root for the approval flow. |
| **Gradebook** | `class_id`, `last_published_at?` | View aggregation; not a write target directly. |
| **ReportCard** | `enrolment_id`, `term_id`, `published_at?`, `approved_by` | Requires leadership approval before publication. |

### Enums

- `AssessmentType`: `assignment`, `quiz`, `exam`.
- `AssessmentStatus`: `draft`, `open`, `closed`, `graded`.
- `GradeStatus`: `draft`, `submitted`, `approved`, `conflict_flagged`.
- `LetterGrade`: per-school configurable scale; default A/B/C/D/E/U.

### State Machine — Grade

```
draft → submitted ──▶ approved
                  └─▶ conflict_flagged → (human resolution) → approved | draft
```

### State Machine — Assessment

```
draft → open → closed → graded
```

---

## Module — Billing

### Entities

| Entity | Key Properties | Notes |
|--------|----------------|-------|
| **FeeHead** | `school_id`, `code`, `name`, `category` | Category enum below. |
| **FeeSchedule** | `fee_head_id`, `term_id`, `amount`, `currency` | Per-term amount. |
| **Invoice** *(root)* | `student_id`, `term_id`, `total`, `balance`, `status`, `issued_at`, `due_at` | Root of Invoice + InvoiceLine + Payment. |
| **InvoiceLine** | `invoice_id`, `fee_head_id`, `amount`, `description` | One line per charged Fee Head. |
| **Payment** | `invoice_id`, `amount`, `method`, `reference`, `paid_at`, `status`, `sync_source` | Mobile-money, cash, bank, Paynow. |
| **Receipt** | `payment_id`, `receipt_number`, `fiscal_reference?`, `issued_at` | `fiscal_reference` set when ZIMRA-fiscalised. |
| **Scholarship** | `student_id`, `fee_head_id?`, `percent`, `amount?`, `term_id`, `approved_by` | Approval required. |
| **Reconciliation** | `period`, `status`, `performed_by`, `completed_at?` | Monthly/term close. |

### Enums

- `FeeCategory`: `tuition`, `boarding`, `transport`, `uniform`, `activity`, `other`.
- `InvoiceStatus`: `draft`, `issued`, `partially_paid`, `paid`, `overdue`, `cancelled`.
- `PaymentMethod`: `cash`, `ecocash`, `onemoney`, `bank_transfer`, `paynow`.
- `PaymentStatus`: `pending`, `confirmed`, `failed`, `refunded`, `conflict_flagged`.

### State Machine — Invoice

```
draft → issued ──┬─▶ partially_paid ─▶ paid
                 ├─▶ paid
                 ├─▶ overdue         (date-driven)
                 └─▶ cancelled
```

---

## Module — Analytics

### Entities

| Entity | Key Properties | Notes |
|--------|----------------|-------|
| **EngagementScore** | `student_id`, `period`, `score`, `inputs_digest` | Derived, recomputed by Celery job. |
| **RiskFlag** | `student_id`, `flag_type`, `severity`, `raised_at`, `acknowledged_by?` | AI-advisory; humans act. |
| **Recommendation** | `target_id`, `target_type`, `kind`, `payload`, `expires_at` | Rule-based in Phase 1. |
| **Dashboard** | `school_id`, `kind`, `config_ref` | Configuration pointer; rendered client-side. |

### Enums

- `RiskFlagType`: `attendance_drop`, `grade_decline`, `fee_default`, `engagement_low`.
- `Severity`: `info`, `watch`, `alert`.

---

## Module — Notifications

### Entities

| Entity | Key Properties | Notes |
|--------|----------------|-------|
| **NotificationTemplate** | `school_id`, `code`, `channel`, `subject_tpl`, `body_tpl` | Channel enum below. |
| **Notification** | `template_id`, `recipient_user_id`, `payload`, `status`, `queued_at`, `delivered_at?` | One record per send attempt. |
| **NotificationPreference** | `user_id`, `channel`, `opted_in` | Per-user opt-in matrix. |

### Enums

- `Channel`: `in_app`, `email`, `sms`, `push`.
- `NotificationStatus`: `queued`, `sending`, `delivered`, `failed`, `cancelled`.

---

## Cross-Cutting Entities

| Entity | Key Properties | Notes |
|--------|----------------|-------|
| **AuditLog** | `school_id`, `actor_user_id`, `action`, `target_type`, `target_id`, `before`, `after`, `at` | Append-only. |
| **SyncRecord** | `device_id`, `user_id`, `entity_type`, `entity_id`, `op`, `client_timestamp`, `server_timestamp`, `status`, `conflict?` | Mobile sync reconciliation. |
| **Attachment** | `owner_type`, `owner_id`, `filename`, `mime`, `size_bytes`, `storage_ref` | Files on local disk; cloud-backup mirror. |

> `FeatureFlag` was previously listed here. It is now owned by the **Schools module** — see Module — Schools above.

---

## Aggregate Boundaries Summary

| Aggregate Root | Children |
|----------------|----------|
| School | SchoolMembership (via Users), FeatureFlag, SchoolConfig |
| User | (global, no children) |
| Student | Enrolment, AttendanceRecord (through Enrolment) |
| Course | Lesson, LessonVersion |
| Assessment | AssessmentItem, Submission |
| Grade | (self) — but state-flow-owning |
| Invoice | InvoiceLine, Payment, Receipt |

Writes to children go through the root's services to enforce invariants defined in [12-BusinessRules.md](12-BusinessRules.md).

---

## Status

**All entities listed here are design-stage.** No tables exist yet. Implementation is tracked in [13-ProjectStatus.md](13-ProjectStatus.md) and scheduled in [11-Roadmap.md](11-Roadmap.md).
