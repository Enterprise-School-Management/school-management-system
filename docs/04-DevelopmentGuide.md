# 04 — Development Guide

Local setup, coding standards, branching, migrations, and contribution workflow for the School Management System.

**See also:** [01-Architecture.md](01-Architecture.md) · [07-DatabaseGuide.md](07-DatabaseGuide.md) · [09-TestingGuide.md](09-TestingGuide.md) · [10-DeploymentGuide.md](10-DeploymentGuide.md)

---

## 1. Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Python | ≥ 3.13 | System package or `pyenv` |
| `uv` | latest | `pip install uv` or see [astral.sh/uv](https://astral.sh/uv) |
| Docker + Compose | latest | Docker Desktop / Engine |
| Node.js | ≥ 20 (LTS) | For the Next.js web client (in a future repo/folder) |
| Git | ≥ 2.40 | System package |

---

## 2. First-Time Setup

```bash
# Clone
git clone <repo-url> school-management-system
cd school-management-system

# Python deps via uv
uv sync

# Apply migrations (SQLite scaffold; switches to Postgres in Phase 1)
uv run python manage.py migrate

# Create a superuser for local admin
uv run python manage.py createsuperuser

# Run dev server
uv run python manage.py runserver
```

The default scaffold uses SQLite (see [07-DatabaseGuide.md](07-DatabaseGuide.md)). PostgreSQL replaces it when the first domain module lands — expect a `.env.example` with `DATABASE_URL` at that point.

---

## 3. Project Layout

Current (scaffold):

```
school-management-system/
├── manage.py
├── pyproject.toml
├── uv.lock
├── db.sqlite3                          # dev-only
├── school_management_system/
│   ├── settings.py
│   ├── urls.py
│   ├── wsgi.py
│   └── asgi.py
├── docs/
│   └── spec.md                         # canonical specification
└── .claude/
    ├── agents/                         # 13 role definitions
    └── docs/                           # this documentation set
```

Target (end of Phase 1):

```
school-management-system/
├── apps/                               # Django apps — one per bounded context
│   ├── users/
│   ├── courses/
│   ├── enrolments/
│   ├── assessments/
│   ├── billing/
│   ├── analytics/
│   └── notifications/
├── core/                               # shared kernel (tenant context, audit helpers)
├── sync/                               # offline sync adapter
├── web/                                # Next.js PWA (or separate repo, TBD)
└── mobile/                             # React Native + Expo (or separate repo, TBD)
```

> The split between monorepo and separate repos for `web/` and `mobile/` is **TBD** — decision tracked in [13-ProjectStatus.md](13-ProjectStatus.md).

---

## 4. Coding Standards

### Python

- **Style:** PEP 8, enforced by Black (line length 100) and Ruff.
- **Type hints:** required on every function signature and dataclass field. Enforced by `mypy --strict` on the domain layer; relaxed on views.
- **Imports:** sorted by `ruff --fix`; stdlib, third-party, first-party groups.
- **Docstrings:** one-line summary on public functions and classes. No decorative comment blocks inside functions.
- **Comments:** only when the *why* is non-obvious. Never restate what the code does.

### Django

- One app per bounded context. Never lump unrelated models into a single app.
- Put business logic in `apps/<mod>/services/*.py`, not in views or model methods.
- Views are thin: parse → call service → serialise → return.
- Never query another app's model directly — call its service or subscribe to its events.

### Naming

- Modules: `snake_case.py`.
- Classes: `PascalCase`.
- Functions / vars: `snake_case`.
- Constants: `UPPER_SNAKE`.
- Test files: `test_<subject>.py`.

### Git commits

- Imperative mood: `Add invoice state machine`, not `added invoice state machine`.
- Reference business-rule codes when relevant: `Enforce BR-INV-005 on cancellation`.
- Conventional Commits optional; consistency of tone matters more than the exact format.

---

## 5. Branching & Pull Requests

### Branch naming

```
feature/<short-description>
fix/<short-description>
chore/<short-description>
docs/<short-description>
```

### PR requirements

- Links to any relevant `BR-*` codes from [12-BusinessRules.md](12-BusinessRules.md).
- Test coverage for the change (see [09-TestingGuide.md](09-TestingGuide.md)).
- No `.env` or secrets in the diff.
- Passes all CI gates (lint, SAST, dependency scan, tests).
- At least one human reviewer; mandatory for AI-assisted changes.

### PR template

```
## What
<one-paragraph summary>

## Why
<business motivation; BR codes if applicable>

## How
<key design choices>

## Tests
<what was added / updated>

## Checklist
- [ ] Migration reversible (if any)
- [ ] Permission matrix updated (if any new endpoint)
- [ ] Docs updated (.claude/docs/ entries that describe the change)
- [ ] No new warnings from ruff/mypy
```

---

## 6. Migrations Workflow

- One migration per logical change.
- Schema and data changes in separate migrations.
- Test every migration against a seeded fixture.
- Breaking changes use the two-phase pattern from [07-DatabaseGuide.md §9](07-DatabaseGuide.md).

```bash
# Create migration
uv run python manage.py makemigrations <app>

# Apply locally
uv run python manage.py migrate

# Dry-run SQL (good for reviews)
uv run python manage.py sqlmigrate <app> NNNN
```

---

## 7. Pre-commit Hooks

**Not yet implemented.** Planned hooks (lands in Phase 0):

- `ruff check --fix`
- `black`
- `mypy` on domain layer
- `bandit` on security-sensitive paths
- `gitleaks` against staged content

Hook config will live in `.pre-commit-config.yaml`.

---

## 8. Environment Variables

Every environment-sensitive value is read from `os.environ`. The repo ships `.env.example` only; real `.env` files are per-install and never committed.

Expected keys (evolving):

```
DJANGO_SECRET_KEY=
DJANGO_DEBUG=false
DJANGO_ALLOWED_HOSTS=school.lan
DATABASE_URL=postgres://...
REDIS_URL=redis://...
JWT_SIGNING_KEY=
PAYNOW_INTEGRATION_ID=
PAYNOW_INTEGRATION_KEY=
SMS_PROVIDER=econet
SMS_API_KEY=
CLOUD_BACKUP_TARGET=
```

---

## 9. Multi-Agent Development Rules

From [docs/spec.md §15.2](../../docs/spec.md):

1. Lock coding standards before code generation begins — this file is the lock.
2. Master system prompt encodes architecture, stack, domain model, coding standards. All AI tools work from it.
3. Generate code from ADRs and API contracts, not vague descriptions.
4. Require tests with every AI-generated module — no exceptions.
5. Run lint, SAST, dependency scan, and human review on all AI-generated code.
6. Keep a small set of approved patterns so agents do not drift.

---

## 10. Local Services

Dev-time services (once the stack lands in Phase 1) run via a project-level `docker-compose.dev.yml`:

```
postgres:  5432
redis:     6379
minio:     9000   # S3-compatible for local cloud-backup testing
mailhog:   8025   # email trap
```

See [10-DeploymentGuide.md](10-DeploymentGuide.md) for production compose.

---

## 11. Common Commands

```bash
# Shell
uv run python manage.py shell

# Make a new app
uv run python manage.py startapp <name> apps/<name>

# Open DB shell
uv run python manage.py dbshell

# Run Celery worker (post-Phase 1)
uv run celery -A school_management_system worker -l info
```

---

## 12. Status

Scaffold-level Django is installed and runnable. App structure, services, tests, and CI gates land during Phase 1. Track progress in [13-ProjectStatus.md](13-ProjectStatus.md).
