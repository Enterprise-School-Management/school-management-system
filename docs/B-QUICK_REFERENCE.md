# B — Quick Reference

At-a-glance cheat sheet for the School Management System. Everything on this page is a summary of something stated more precisely elsewhere — follow the link when you need the full picture.

**See also:** [A-INDEX.md](A-INDEX.md) · [C-Glossary.md](C-Glossary.md) · [04-DevelopmentGuide.md](04-DevelopmentGuide.md)

---

## Principles in One Line Each

- Offline-first everywhere — the school server is authoritative; internet is optional.
- Modular monolith first — services only when scale demands them.
- Multi-tenant via row-level `school_id`.
- Mobile (Android + iOS) is first-class, not deferred.
- AI is advisory — humans make every final decision.
- Plugin-activated modules — tiered pricing without code changes.
- Zimbabwe context — ZESA outages, LAN-only, low-end Android, local SMS/payments.

Detail: [01-Architecture.md](01-Architecture.md).

---

## Tech Stack at a Glance

| Layer | Choice |
|-------|--------|
| Web | Next.js (App Router) + PWA |
| Mobile | React Native + Expo + WatermelonDB |
| Backend | Django + DRF (ASGI-ready) |
| DB | PostgreSQL 16 (+ pgvector) |
| Cache / broker | Redis |
| Queue | Celery |
| Auth | Django Auth + JWT |
| Packaging | Docker + Docker Compose |
| CI | GitHub Actions |

Detail: [01-Architecture.md §8](01-Architecture.md).

---

## Domain Modules

`Users` · `Courses` · `Enrolments` · `Assessments` · `Billing` · `Analytics` · `Notifications`

Detail: [01-Architecture.md §3](01-Architecture.md), [06-DomainModel.md](06-DomainModel.md).

---

## The 13 Roles

| Group | Role codes |
|-------|-----------|
| Leadership | `school_head` |
| Operations | `registrar` · `examinations_officer` · `finance_bursar` |
| Teaching / support | `teacher` · `lms_specialist` · `hostel_parent` · `school_psychologist` |
| Users | `student` · `parent_guardian` |
| Quality / tech | `qa_engineer` · `security_officer` · `system_designer` |

Detail: [08-SecurityGuide.md §2](08-SecurityGuide.md), [05-TrainingManual.md](05-TrainingManual.md).

---

## Common Commands

```bash
# Environment
uv sync                                 # install deps
uv run python manage.py runserver       # dev server
uv run python manage.py migrate         # apply migrations
uv run python manage.py createsuperuser # first admin
uv run python manage.py shell           # ORM REPL

# Migrations (post-Phase 1)
uv run python manage.py makemigrations <app>
uv run python manage.py sqlmigrate <app> NNNN

# Tests (post-Phase 1)
uv run pytest
uv run pytest --cov --cov-report=term-missing

# Docker (post-Phase 0)
docker compose up -d
docker compose exec web python manage.py migrate
docker compose pull && docker compose up -d   # upgrade
```

Detail: [04-DevelopmentGuide.md](04-DevelopmentGuide.md), [10-DeploymentGuide.md](10-DeploymentGuide.md).

---

## Local URLs

| What | URL |
|------|-----|
| Dev API | `http://127.0.0.1:8000/` |
| Django admin | `http://127.0.0.1:8000/admin/` |
| API root | `http://127.0.0.1:8000/api/v1/` |
| Health | `http://127.0.0.1:8000/api/v1/health` |
| Mailhog (dev) | `http://127.0.0.1:8025/` |
| MinIO (dev) | `http://127.0.0.1:9000/` |

Production (school LAN): `https://school.lan/` with self-signed CA distributed to devices.

---

## API Conventions

- Base path: `/api/v1/`.
- Auth: `Authorization: Bearer <jwt>`.
- Tenant: embedded in JWT — never passed as a parameter.
- Pagination: `?page=1&page_size=25` (max 100).
- Sort: `?sort=-created_at`.
- Idempotency: `Idempotency-Key` header — required on `POST /payments`.

Detail: [03-ApiReference.md](03-ApiReference.md).

---

## State Machines

```
Enrolment:   active → withdrawn | graduated | transferred
Assessment:  draft → open → closed → graded
Grade:       draft → submitted → approved
                           ↘ conflict_flagged → draft | approved
Invoice:     draft → issued → partially_paid → paid
                           → overdue | cancelled
```

Detail: [06-DomainModel.md](06-DomainModel.md).

---

## Conflict Resolution

| Data | Strategy |
|------|----------|
| Grades | Conflict-flag; human resolution |
| Payments | Conflict-flag; human resolution |
| Everything else | Last-write-wins |

Detail: [01-Architecture.md §7](01-Architecture.md).

---

## Branch & Commit Conventions

```
feature/<short-description>
fix/<short-description>
chore/<short-description>
docs/<short-description>
```

- Imperative commit subject: `Add invoice state machine`.
- Reference `BR-*` codes where relevant.
- PRs require tests, no secrets, CI green, one human reviewer (mandatory for AI-assisted changes).

Detail: [04-DevelopmentGuide.md §5](04-DevelopmentGuide.md).

---

## Zimbabwe Context Constants

| Concern | Providers / rails |
|---------|-------------------|
| Mobile money | EcoCash · OneMoney |
| Payment gateway | Paynow — [developers.paynow.co.zw](https://developers.paynow.co.zw/docs/initiate_mobile_transaction.htm) |
| SMS gateways | Econet · NetOne |
| Fiscal compliance | ZIMRA fiscal device |
| Data protection | Cyber Security and Data Protection Act (DPA) |
| Ministry oversight | Ministry of Primary and Secondary Education |

Detail: [C-Glossary.md](C-Glossary.md), [08-SecurityGuide.md](08-SecurityGuide.md), [12-BusinessRules.md](12-BusinessRules.md).

---

## CI Security Gates

`lint` · `SAST (Bandit / Semgrep)` · `dependency scan (pip-audit / npm audit)` · `container scan (Trivy)` · `secrets scan (gitleaks)` · `human review`.

Any failure blocks merge. Detail: [08-SecurityGuide.md §9](08-SecurityGuide.md).

---

## Documentation Conventions

- `TBD` = design not settled.
- `Not yet implemented` = design settled, code pending.
- Dates: `YYYY-MM-DD`.
- File refs: markdown links with relative paths.
- No emojis.

---

## When Stuck

1. [A-INDEX.md](A-INDEX.md) — find the right doc.
2. [C-Glossary.md](C-Glossary.md) — decode the term.
3. [docs/spec.md](../../docs/spec.md) — source of truth.
4. Relevant role under [.claude/agents/](../agents/) — perspective check.
