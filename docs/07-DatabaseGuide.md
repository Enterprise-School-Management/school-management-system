# 07 — Database Guide

Schema conventions, naming rules, migration workflow, and operational practices for the PostgreSQL database.

**See also:** [06-DomainModel.md](06-DomainModel.md) · [01-Architecture.md](01-Architecture.md) · [10-DeploymentGuide.md](10-DeploymentGuide.md)

---

## 1. Engine

| Environment | Engine | Rationale |
|-------------|--------|-----------|
| Production (school server) | PostgreSQL 16+ | Canonical, with `pgvector` extension for AI features. |
| Development | PostgreSQL 16+ via Docker | Mirror production. |
| CI | PostgreSQL 16+ service | Never SQLite for integration tests — avoid behaviour drift. |

> **Current code-state caveat:** The initial Django scaffold ships with SQLite in [school_management_system/settings.py](../../school_management_system/settings.py). This is a scaffold default only. PostgreSQL is the target engine from first feature work — update `DATABASES` when the first module lands. Tracked in [13-ProjectStatus.md](13-ProjectStatus.md).

---

## 2. Table Naming

- `snake_case`, lowercase.
- Plural nouns for tables (`students`, `invoices`, `attendance_records`).
- Join tables: alphabetical concatenation (`guardians_students`).
- History/audit tables: suffix `_audit` or `_history`.
- Keep names within 30 characters where possible for PostgreSQL tooling comfort.

---

## 3. Column Conventions

### Universal columns (every table)

| Column | Type | Notes |
|--------|------|-------|
| `id` | `uuid` (v7 if available, else v4) | Primary key. |
| `created_at` | `timestamptz` | Default `now()`. |
| `updated_at` | `timestamptz` | Touched on every write. |
| `deleted_at` | `timestamptz NULL` | Soft delete. Queries default to `WHERE deleted_at IS NULL`. |

### Tenant-scoped columns

Every table except `schools`, `users`, and global lookups adds:

| Column | Type | Notes |
|--------|------|-------|
| `school_id` | `uuid NOT NULL` | FK → `schools.id`. Indexed. |

### Auditable writes

Business-critical tables (grades, invoices, payments, students) add:

| Column | Type | Notes |
|--------|------|-------|
| `created_by` | `uuid NULL` | FK → `users.id`. |
| `updated_by` | `uuid NULL` | FK → `users.id`. |

---

## 4. Foreign Keys

- Always declare FK constraints; never emulate via app-layer checks.
- Default action: `ON DELETE RESTRICT`. Cascade only where the parent-child relationship is total (e.g. `invoices → invoice_lines`).
- Tenant FK (`school_id`) is `ON DELETE RESTRICT` always — a school cannot be deleted while any child row exists.
- `users.id` FKs use `ON DELETE SET NULL` for `created_by` / `updated_by` so deleting a user doesn't cascade-destroy history. Soft delete is preferred.

---

## 5. Indexes

Create indexes for:

1. **Every FK** — PostgreSQL does not auto-index FKs.
2. **`school_id`** on every tenant-scoped table (queried on every request).
3. **Composite `(school_id, <frequently-filtered-column>)`** — e.g. `(school_id, status)` on invoices, `(school_id, date)` on attendance.
4. **Unique constraints** — e.g. `(school_id, student_number)`, `(school_id, invoice_number)`.
5. **Search-targeted columns** — use `pg_trgm` GIN for name searches.

Avoid indexing low-selectivity booleans alone.

---

## 6. Multi-Tenancy Enforcement

Row-level `school_id` filtering is enforced at three points:

1. **ORM manager** — default queryset wraps every read/write with a tenant filter from request context.
2. **Integration tests** — a tenant-leak suite attempts cross-tenant access and must fail; see [09-TestingGuide.md](09-TestingGuide.md).
3. **PostgreSQL Row-Level Security (RLS)** — planned as a defence-in-depth layer (**Not yet implemented**). Enable once the tenant-context session variable plumbing is in place.

---

## 7. Soft Delete

- Set `deleted_at = now()`; never issue `DELETE` for application data.
- Default querysets exclude soft-deleted rows.
- Restore is a simple `UPDATE deleted_at = NULL` by authorised roles.
- Hard delete is reserved for GDPR/DPA right-to-erasure requests; tracked via a separate admin flow and audit entry.

---

## 8. Audit Logging

- Append-only `audit_logs` table (see [06-DomainModel.md](06-DomainModel.md) cross-cutting entities).
- Every write service emits one audit entry inside the same DB transaction as the write.
- `before` and `after` stored as `jsonb`.
- `audit_logs` is never soft-deleted and never edited.

---

## 9. Migrations

- Framework: Django migrations.
- One migration per logical change; do not squash until a release boundary.
- Naming: `NNNN_<short_verb>_<subject>.py` (e.g. `0007_add_conflict_flag_to_grades.py`).
- Every migration must be tested in CI against a populated fixture set.
- Breaking changes require a two-phase migration: (1) add new column / backfill, (2) switch reads, (3) drop old column in a later release.

### Schema change checklist

- [ ] Forward migration written and reversible where possible.
- [ ] Data migration separated from schema migration.
- [ ] Indexes created `CONCURRENTLY` in production migrations on tables > 100k rows.
- [ ] Tenant-scope column added if applicable.
- [ ] Audit-log impact considered.
- [ ] Test fixtures updated.

---

## 10. Seeding

Two seed levels:

| Level | Purpose | Lives In |
|-------|---------|----------|
| **System seed** | Roles, fee categories, notification channels — facts every install needs. | Migrations marked `data=True`. |
| **Demo seed** | One fictional school with sample users, classes, invoices — for local dev only. | Management command `seed_demo`. Never runs in production. |

---

## 11. Backup & Recovery

- **Local backup:** `pg_dump` to a separate disk on the school server, nightly, retained 30 days.
- **Cloud backup:** Opportunistic upload when internet is available — encrypted at rest.
- **Point-in-time recovery:** WAL archiving to the backup disk. RPO target: 15 minutes. RTO target: 2 hours.
- **Restore drills:** Quarterly, per [09-TestingGuide.md](09-TestingGuide.md).

---

## 12. Performance

- Expected working-set: 500–5,000 students per school. One school per server is common.
- Connection pooling via `pgbouncer` in production; **Not yet implemented** in scaffold.
- Slow-query logging enabled (> 200 ms).
- Regular `VACUUM ANALYZE` is the autovacuum default; monitored in production.

---

## 13. Status

All conventions above are design-stage. No production tables exist yet. Adoption begins with the first feature migration in Phase 1 (see [11-Roadmap.md](11-Roadmap.md)).
