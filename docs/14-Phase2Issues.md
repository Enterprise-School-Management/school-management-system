# 14 — Phase 2 GitHub Issues

Atomic GitHub issues for Phase 2 of the school management system build.

**Phase 1 covered:** Schools module discovery, School entity definition, TenantContext, School model implementation, module activation flags, cross-module event contracts, and superadmin API.

**Phase 2 covers:**
- Docker and local dev environment setup (Postgres, Redis, Celery, Django in one Compose file)
- Base Django project structure and module skeleton pattern
- Users module (auth, roles, permissions, JWT, relationship to Schools)
- CI/CD scaffold (GitHub Actions — lint, test, build)

**See also:** [01-Architecture.md](01-Architecture.md) · [06-DomainModel.md](06-DomainModel.md) · [04-DevelopmentGuide.md](04-DevelopmentGuide.md) · [08-SecurityGuide.md](08-SecurityGuide.md) · [11-Roadmap.md](11-Roadmap.md) · [13-ProjectStatus.md](13-ProjectStatus.md)

---

## Issue format

Each issue uses the format `[Module] Title` and includes:
- **Summary** — one-paragraph context and motivation.
- **Work to do** — concrete, implementation-level tasks.
- **Acceptance criteria** — verifiable exit conditions.
- **Blocked by** — prerequisite issues (by title or number).

---

## Area 1 — Docker and Local Dev Environment

---

### [DevOps] Add Docker Compose dev stack (Postgres, Redis, Celery, Django)

**Summary**

The project currently has no container orchestration. Per [01-Architecture.md §9](01-Architecture.md) and the Phase 0 deliverables in [11-Roadmap.md](11-Roadmap.md), all services (Django, PostgreSQL 16, Redis, Celery worker) must run via a single `docker compose up` command. This is the foundation every other Phase 2 issue depends on.

**Work to do**

- Create `docker-compose.dev.yml` at the repo root with the following named services:
  - `db` — `postgres:16-alpine`, persistent volume `pgdata`, exposed on `5432`.
  - `redis` — `redis:7-alpine`, exposed on `6379`.
  - `web` — Django dev server (`uv run python manage.py runserver 0.0.0.0:8000`), mounts repo as volume, depends on `db` and `redis`.
  - `worker` — Celery worker (`uv run celery -A school_management_system worker -l info`), same image as `web`, depends on `db` and `redis`.
  - `mailhog` — `mailhog/mailhog`, SMTP trap on port `1025`, web UI on `8025`.
- Write a `Dockerfile` (dev target) for the Django/Celery image based on `python:3.13-slim`; install dependencies via `uv sync`.
- Add `.dockerignore` excluding `.git`, `__pycache__`, `*.pyc`, `db.sqlite3`, `.env`.
- Document the `docker compose -f docker-compose.dev.yml up` one-liner in [04-DevelopmentGuide.md §2](04-DevelopmentGuide.md).

**Acceptance criteria**

- `docker compose -f docker-compose.dev.yml up` starts all services without errors.
- `curl http://localhost:8000/` returns an HTTP response from Django.
- Celery worker logs "ready" after startup.
- MailHog web UI is reachable at `http://localhost:8025`.
- No hard-coded secrets in `docker-compose.dev.yml` — all sensitive values read from a `.env` file.

**Blocked by**

- Nothing (first issue in Phase 2).

---

### [DevOps] Create .env.example and wire settings.py to environment variables

**Summary**

`settings.py` currently contains hard-coded values (SQLite path, `DEBUG=True`, placeholder `SECRET_KEY`). Per [04-DevelopmentGuide.md §8](04-DevelopmentGuide.md) and [08-SecurityGuide.md §5](08-SecurityGuide.md), every environment-sensitive value must be read from `os.environ`. The repo ships `.env.example` only; real `.env` files are per-install and never committed.

**Work to do**

- Add `django-environ` or `python-decouple` to `pyproject.toml` dependencies.
- Create `.env.example` at the repo root with all required keys and safe placeholder values:
  ```
  DJANGO_SECRET_KEY=change-me-in-production
  DJANGO_DEBUG=true
  DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
  DATABASE_URL=postgres://sms:sms@db:5432/sms
  REDIS_URL=redis://redis:6379/0
  JWT_SIGNING_KEY=change-me-in-production
  EMAIL_HOST=mailhog
  EMAIL_PORT=1025
  ```
- Update `school_management_system/settings.py` to read every value from the environment with safe defaults for local dev.
- Add `.env` to `.gitignore` (verify it is not already there).
- Add a note to [04-DevelopmentGuide.md §2](04-DevelopmentGuide.md): copy `.env.example` to `.env` as the first-time setup step.

**Acceptance criteria**

- `cp .env.example .env && docker compose -f docker-compose.dev.yml up` starts the stack with no additional configuration.
- `settings.py` contains no hard-coded secrets or database paths.
- `git diff --name-only` never includes `.env` in committed output.
- CI passes without a `.env` file present (values injected via GitHub Actions secrets).

**Blocked by**

- `[DevOps] Add Docker Compose dev stack (Postgres, Redis, Celery, Django)`

---

### [DevOps] Switch development database from SQLite to PostgreSQL

**Summary**

The Django scaffold defaults to SQLite. Per [13-ProjectStatus.md](13-ProjectStatus.md) locked decision #2, PostgreSQL 16 is the production and development database; SQLite is scaffold noise to be removed. Per [09-TestingGuide.md §2](09-TestingGuide.md), integration tests must never run against SQLite — behaviour drifts from Postgres. This issue replaces the default `DATABASES` configuration and removes `db.sqlite3`.

**Work to do**

- Remove `db.sqlite3` from the repo (add to `.gitignore` if not already present).
- Update `DATABASES` in `settings.py` to parse `DATABASE_URL` using `dj-database-url` or `django-environ`.
- Add `psycopg[binary]` (or `psycopg2-binary` for compatibility) to `pyproject.toml`.
- Run `uv run python manage.py migrate` against the Dockerised Postgres and verify it completes cleanly.
- Update [04-DevelopmentGuide.md §2](04-DevelopmentGuide.md) first-time setup steps to reference Postgres.
- Update [07-DatabaseGuide.md](07-DatabaseGuide.md) status entry for Postgres to **Complete**.

**Acceptance criteria**

- `uv run python manage.py migrate` succeeds when `DATABASE_URL` points to Postgres (via Docker Compose).
- No SQLite file is created or referenced outside of test-only override configs.
- `uv run python manage.py check --database default` passes with the Postgres connection.
- CI test job uses a Postgres service container, not SQLite.

**Blocked by**

- `[DevOps] Add Docker Compose dev stack (Postgres, Redis, Celery, Django)`
- `[DevOps] Create .env.example and wire settings.py to environment variables`

---

### [DevOps] Add pre-commit hooks (ruff, black, mypy, bandit, gitleaks)

**Summary**

Per [04-DevelopmentGuide.md §7](04-DevelopmentGuide.md), pre-commit hooks enforce code quality at commit time and prevent secret leakage before any code reaches CI. This issue creates `.pre-commit-config.yaml` with the planned hook set.

**Work to do**

- Add `pre-commit` to dev dependencies in `pyproject.toml`.
- Create `.pre-commit-config.yaml` with the following hooks:
  - `ruff` — linter and autofix (`ruff check --fix`).
  - `black` — formatter (line length 100, matching [04-DevelopmentGuide.md §4](04-DevelopmentGuide.md)).
  - `mypy` — strict type-checking on `core/` and `apps/*/domain/` (domain layer only per [01-Architecture.md §6](01-Architecture.md) rule 1).
  - `bandit` — SAST on security-sensitive paths.
  - `gitleaks` — scan staged content for secrets.
- Document `pre-commit install` as a first-time setup step in [04-DevelopmentGuide.md §2](04-DevelopmentGuide.md).
- Update [13-ProjectStatus.md](13-ProjectStatus.md) pre-commit row to **Complete**.

**Acceptance criteria**

- `pre-commit run --all-files` passes on a clean checkout.
- A commit containing a `DEBUG=True`-leaked secret string is blocked by gitleaks.
- ruff and black hooks auto-fix trivial style issues on commit.
- mypy hook fails on a function without a return-type annotation in the domain layer.

**Blocked by**

- `[DevOps] Create .env.example and wire settings.py to environment variables`

---

## Area 2 — Base Django Project Structure and Module Skeleton Pattern

---

### [Core] Scaffold apps/ directory with bounded-context app skeletons

**Summary**

Per [04-DevelopmentGuide.md §3](04-DevelopmentGuide.md) (target layout) and [01-Architecture.md §5](01-Architecture.md), each domain module lives as a self-contained Django app under `apps/`. Per [01-Architecture.md §6](01-Architecture.md) (Dependency Rules), every app follows the DDD + Hexagonal skeleton: `domain/` (entities and services), `infrastructure/` (repositories and ORM models), `api/` (DRF views and serializers), and `tests/`. This issue creates the skeleton for all seven bounded contexts.

**Work to do**

- Create `apps/` directory.
- For each of the seven modules (`users`, `courses`, `enrolments`, `assessments`, `billing`, `analytics`, `notifications`):
  - Run `uv run python manage.py startapp <name> apps/<name>`.
  - Create sub-directories: `domain/`, `infrastructure/`, `api/`, `tests/unit/`, `tests/integration/`, `tests/fixtures/`.
  - Add stub `__init__.py` in each sub-directory.
  - Add the app to `INSTALLED_APPS` in `settings.py` as `apps.<name>`.
- Create a `conftest.py` under `apps/` that configures pytest-django settings.
- Document the skeleton pattern (directory map) in [04-DevelopmentGuide.md §3](04-DevelopmentGuide.md).

**Acceptance criteria**

- `uv run python manage.py check` passes with all seven apps registered.
- Each app directory contains `domain/`, `infrastructure/`, `api/`, `tests/` sub-directories.
- `pytest --collect-only` discovers the `tests/` directories for all seven apps.
- No business logic lives in `models.py` or `views.py` stubs (they remain empty or import only from their own sub-packages).

**Blocked by**

- `[DevOps] Switch development database from SQLite to PostgreSQL`

---

### [Core] Create core/ shared kernel — TenantContext, base ORM manager, and AuditLog model

**Summary**

Per [01-Architecture.md §6](01-Architecture.md) (Dependency Rule 4), the shared kernel must be small — only truly universal concepts. This issue creates `core/` with three cross-cutting items: `TenantContext` (request-scoped `school_id` resolution), `TenantManager` (base queryset that auto-injects the `school_id` filter), and the `AuditLog` model (append-only write record for every mutation). These are prerequisites for every domain module. See [06-DomainModel.md — Cross-Cutting Entities](06-DomainModel.md) and [08-SecurityGuide.md §4](08-SecurityGuide.md).

**Work to do**

- Create `core/` as a Django app (`uv run python manage.py startapp core core/`); add to `INSTALLED_APPS`.
- Implement `core/tenant.py`:
  - `TenantContext` — a thread-local or context-var holder that stores the request's `school_id` (set by middleware, cleared after request).
  - `TenantManager` — a Django model manager whose `get_queryset()` auto-filters by `TenantContext.school_id`; raises `ImproperlyConfigured` if called outside a tenant context.
  - `TenantModel` — abstract base Django model (with `school_id` FK to `schools.School` + `created_at` / `updated_at` / `deleted_at` fields) that uses `TenantManager` as default manager. Matches [06-DomainModel.md — Conventions](06-DomainModel.md).
- Implement `core/models.py` — `AuditLog` model per [06-DomainModel.md — Cross-Cutting Entities](06-DomainModel.md):
  - Fields: `school_id`, `actor_user_id`, `action`, `target_type`, `target_id`, `before` (JSONField), `after` (JSONField), `ip_address`, `user_agent`, `at`.
  - Meta: `managed = True`, `ordering = ["-at"]`, no soft-delete (append-only).
- Implement `core/audit.py` — `emit_audit_log(actor, action, target, before, after, request)` helper used in all write paths.
- Create initial migration.
- Write unit tests in `core/tests/unit/test_tenant.py` covering: queryset filters by school_id; raises outside context; cross-tenant query blocked.

**Acceptance criteria**

- Migration runs cleanly on Postgres.
- `TenantManager` queryset always contains only rows matching the current `TenantContext.school_id`.
- Accessing a queryset with `TenantManager` outside a tenant context raises an explicit error.
- `emit_audit_log` writes exactly one `AuditLog` row in the same database transaction as the wrapped write.
- All unit tests in `core/tests/` pass.

**Blocked by**

- `[Core] Scaffold apps/ directory with bounded-context app skeletons`

---

### [Core] Create sync/ offline-sync adapter skeleton and SyncRecord model

**Summary**

Per [01-Architecture.md §8](01-Architecture.md) (Offline-First Strategy) and [06-DomainModel.md — Cross-Cutting Entities](06-DomainModel.md), the `SyncRecord` entity tracks mobile device queue entries, replay state, and conflict flags. This issue creates the `sync/` module skeleton and `SyncRecord` model so that later mobile-sync work has a stable foundation. No sync logic is implemented here — only the structural scaffold.

**Work to do**

- Create `sync/` as a Django app; add to `INSTALLED_APPS`.
- Implement `sync/models.py` — `SyncRecord` model per [06-DomainModel.md](06-DomainModel.md):
  - Fields: `device_id`, `user_id` (FK User), `entity_type`, `entity_id` (UUID), `op` (enum: `create`, `update`, `delete`), `client_timestamp`, `server_timestamp`, `status` (enum: `pending`, `applied`, `conflicted`, `rejected`), `conflict` (JSONField, nullable), `school_id`.
  - `school_id` uses `TenantModel` base from `core/`.
- Stub `sync/reconciler.py` with `reconcile(record: SyncRecord) -> None` — body is `raise NotImplementedError`.
- Stub `sync/device_queue.py` with `enqueue(payload: dict) -> SyncRecord` — body is `raise NotImplementedError`.
- Create initial migration.
- Document sync skeleton entry in [13-ProjectStatus.md](13-ProjectStatus.md) as **Partial**.

**Acceptance criteria**

- Migration runs cleanly on Postgres.
- `SyncRecord` inherits from `TenantModel` — its queryset is automatically `school_id`-scoped.
- `from sync.reconciler import reconcile` imports without error.
- `from sync.device_queue import enqueue` imports without error.
- No sync logic exists yet — stubs raise `NotImplementedError` explicitly.

**Blocked by**

- `[Core] Create core/ shared kernel — TenantContext, base ORM manager, and AuditLog model`

---

## Area 3 — Users Module

---

### [Users] Implement User and School Django models

**Summary**

`User` and `School` are the two top-level entities of the Users module per [06-DomainModel.md — Module: Users](06-DomainModel.md). `School` is the top-level tenant root (no `school_id` FK on itself). `User` is a global identity that may belong to multiple schools. This issue creates both models, their migrations, and their factory fixtures — before any auth or RBAC logic is added.

**Work to do**

- In `apps/users/infrastructure/models.py`:
  - Implement `School` model: fields `id` (UUID PK), `name`, `short_code` (unique), `address`, `timezone`, `active_modules` (ArrayField or JSONField), `created_at`, `updated_at`, `deleted_at` (nullable). No `school_id` FK — it is the tenant root. Matches [06-DomainModel.md](06-DomainModel.md).
  - Implement `User` model extending `AbstractBaseUser` + `PermissionsMixin`: fields `id` (UUID PK), `email` (unique, login field), `full_name`, `phone` (nullable), `is_active`, `last_login_at`, `created_at`, `updated_at`. Custom `UserManager`.
- Register `AUTH_USER_MODEL = "users.User"` in `settings.py`.
- Create initial migration.
- Add `apps/users/tests/fixtures/factories.py` with `SchoolFactory` and `UserFactory` (using `factory_boy`).
- Write unit tests in `apps/users/tests/unit/test_models.py`: School short_code uniqueness, User email uniqueness, soft-delete field nullable.

**Acceptance criteria**

- Migration runs cleanly on Postgres from a blank database.
- `User` and `School` match every field in [06-DomainModel.md — Module: Users](06-DomainModel.md).
- `SchoolFactory` and `UserFactory` produce valid instances without database errors.
- Unit tests pass.
- `uv run python manage.py check` passes.

**Blocked by**

- `[Core] Create core/ shared kernel — TenantContext, base ORM manager, and AuditLog model`

---

### [Users] Implement SchoolMembership, Role, and Guardian models

**Summary**

`SchoolMembership` binds a `User` to a `School` with a `Role`. `Role` carries a `RoleCode` enum covering all 13 roles defined in [06-DomainModel.md](06-DomainModel.md) and [08-SecurityGuide.md §2](08-SecurityGuide.md). `Guardian` links a user to one or more students. These entities complete the Users module data model before auth and RBAC are layered on.

**Work to do**

- In `apps/users/infrastructure/models.py`:
  - Implement `RoleCode` as a `TextChoices` enum with all 13 values from [06-DomainModel.md — Enums](06-DomainModel.md): `school_head`, `registrar`, `examinations_officer`, `finance_bursar`, `teacher`, `lms_specialist`, `hostel_parent`, `school_psychologist`, `qa_engineer`, `security_officer`, `system_designer`, `student`, `parent_guardian`.
  - Implement `Role` model: `code` (RoleCode, unique), `name`, `permissions` (JSONField list of strings).
  - Implement `SchoolMembership` model (extends `TenantModel`): `user` (FK User), `school` (FK School), `role` (FK Role), `started_at`, `ended_at` (nullable). Unique together: `(user, school)`.
  - Implement `Guardian` model (extends `TenantModel`): `user` (FK User), `relationship`, `linked_students` (M2M to a placeholder `Student` model stub — real Student lands in Enrolments module).
- Create migration.
- Add `SchoolMembershipFactory`, `RoleFactory`, `GuardianFactory` to `apps/users/tests/fixtures/factories.py`.
- Unit tests: all 13 `RoleCode` values present; `SchoolMembership` unique constraint enforced; `Guardian` FK resolves.

**Acceptance criteria**

- Migration runs cleanly on Postgres.
- All 13 `RoleCode` values match [06-DomainModel.md](06-DomainModel.md) and [08-SecurityGuide.md §2](08-SecurityGuide.md) exactly.
- `SchoolMembership` with duplicate `(user, school)` raises `IntegrityError`.
- Factory fixtures create valid instances.
- Unit tests pass.

**Blocked by**

- `[Users] Implement User and School Django models`

---

### [Users] Implement JWT authentication with refresh token rotation

**Summary**

Per [08-SecurityGuide.md §1](08-SecurityGuide.md), authentication uses Django Auth + JWT with two token types: access (15 min) and refresh (7 days). Refresh returns a new token pair; the old refresh token is revoked. JWT claims include `user_id`, `school_id` (active membership), `role`, `issued_at`, `exp`, `jti`. The implementation uses `djangorestframework-simplejwt`.

**Work to do**

- Add `djangorestframework-simplejwt` to `pyproject.toml`.
- Configure `SIMPLE_JWT` in `settings.py`:
  - Access token lifetime: 15 minutes.
  - Refresh token lifetime: 7 days.
  - Token rotation: `ROTATE_REFRESH_TOKENS = True`, `BLACKLIST_AFTER_ROTATION = True`.
  - Add `rest_framework_simplejwt.token_blacklist` to `INSTALLED_APPS`.
  - Signing key: read from `JWT_SIGNING_KEY` env var.
- Implement a custom `TokenObtainPairSerializer` in `apps/users/api/serializers.py` that injects `school_id` (active `SchoolMembership`) and `role` into token claims.
- Wire auth endpoints in `school_management_system/urls.py`:
  - `POST /api/v1/auth/login` — obtain token pair.
  - `POST /api/v1/auth/refresh` — rotate token pair.
  - `POST /api/v1/auth/logout` — blacklist refresh token.
- Create migration for token blacklist tables.
- Integration tests in `apps/users/tests/integration/test_auth.py`:
  - Login returns valid access + refresh tokens.
  - Refresh rotates the pair and revokes the old refresh token.
  - Using a revoked refresh token returns 401.
  - Claims contain `school_id` and `role`.

**Acceptance criteria**

- `POST /api/v1/auth/login` returns `{ "access": "…", "refresh": "…" }` with correct claim set.
- `POST /api/v1/auth/refresh` returns a new pair; the old refresh token is rejected on re-use with 401.
- `POST /api/v1/auth/logout` blacklists the refresh token; subsequent refresh returns 401.
- JWT claims include `user_id`, `school_id`, `role`, `exp`, `jti`.
- All integration tests pass.

**Blocked by**

- `[Users] Implement SchoolMembership, Role, and Guardian models`

---

### [Users] Implement RBAC permission classes covering the 08-SecurityGuide.md matrix

**Summary**

Per [01-Architecture.md §6](01-Architecture.md) (Dependency Rules) and [08-SecurityGuide.md §2–§3](08-SecurityGuide.md), every DRF endpoint must enforce: (1) valid JWT, (2) role permission check against the matrix, (3) row-scope check for `(own)` / `(self)` / `(linked)` modifiers. This issue creates reusable DRF permission classes so every subsequent module can use them with a one-liner decorator.

**Work to do**

- Implement `core/permissions.py`:
  - `IsAuthenticated` — requires valid JWT (standard, re-exported).
  - `HasRole(*role_codes)` — DRF permission class that checks `request.auth["role"]` against the provided `RoleCode` list.
  - `IsTenantMember` — verifies the JWT's `school_id` matches an active `SchoolMembership`.
  - `IsOwnerOrRole(owner_field, *role_codes)` — row-scope check: passes if `record.<owner_field> == request.user` or role is in `role_codes`.
- Implement `core/decorators.py` — `require_roles(*role_codes)` view decorator that composes `IsAuthenticated + IsTenantMember + HasRole` for convenience.
- Unit tests in `core/tests/unit/test_permissions.py` generated from the permission matrix in [08-SecurityGuide.md §2](08-SecurityGuide.md):
  - One test per critical role × endpoint pattern (at minimum: `school_head`, `teacher`, `student`, `parent_guardian`).
  - Test that a role without access receives 403.
  - Test that an unauthenticated request receives 401.

**Acceptance criteria**

- `HasRole("school_head")` on an endpoint returns 403 for a `teacher`-role JWT.
- `IsTenantMember` rejects a JWT whose `school_id` is not the authenticated user's active membership.
- `IsOwnerOrRole("created_by", "school_head")` returns 403 when a `teacher` attempts to access another teacher's resource.
- All unit tests in `core/tests/unit/test_permissions.py` pass.
- `uv run python manage.py check` passes.

**Blocked by**

- `[Users] Implement JWT authentication with refresh token rotation`
- `[Core] Create core/ shared kernel — TenantContext, base ORM manager, and AuditLog model`

---

### [Users] Implement tenant-scoping middleware and ORM manager

**Summary**

Per [01-Architecture.md §7](01-Architecture.md) (Multi-Tenancy Enforcement) and [08-SecurityGuide.md §3](08-SecurityGuide.md) (Scoping Layers), every request must pass through a tenant-scoping layer that: (1) resolves `school_id` from the JWT claims, (2) loads it into `TenantContext`, and (3) allows the `TenantManager` to auto-filter all ORM queries. Cross-tenant access (superadmin) goes through a separate explicit API that is never reachable from the regular middleware path. This is the foundational security control for multi-tenancy and must be verified by a dedicated regression suite.

**Work to do**

- Implement `core/middleware.py` — `TenantMiddleware(get_response)`:
  - On each request: decode the JWT (if present), extract `school_id` from claims, set `TenantContext.school_id`.
  - On response return: clear `TenantContext.school_id` to prevent context bleed across requests.
  - Unauthenticated requests: set `TenantContext.school_id = None` (public endpoints and login pass through).
- Wire `TenantMiddleware` into `settings.py` `MIDDLEWARE` list after Django's `AuthenticationMiddleware`.
- Verify `TenantManager` (from `[Core] Create core/ shared kernel`) raises `TenantContextNotSet` if `school_id` is `None` and a tenant-scoped queryset is evaluated.
- Add a superadmin bypass: a JWT with `role = "superadmin"` and no `school_id` claim sets `TenantContext.superadmin = True`; a separate `CrossTenantManager` is used for those queries only.
- Write a dedicated tenant-isolation test suite in `core/tests/integration/test_tenant_isolation.py`:
  - Create two schools, two users (one per school).
  - For every CRUD operation on a tenant-scoped model: assert School A user cannot read School B data (expects 404 or empty queryset).
  - Assert superadmin can read across schools using the explicit cross-tenant path.

**Acceptance criteria**

- `TenantMiddleware` correctly sets and clears `TenantContext` on every request/response cycle.
- A user authenticated to School A receives an empty queryset (not School B data) when querying any tenant-scoped model.
- Evaluating a `TenantManager` queryset with `TenantContext.school_id = None` raises `TenantContextNotSet`.
- Tenant-isolation test suite passes with zero cross-tenant leaks.
- `uv run python manage.py check` passes.

**Blocked by**

- `[Users] Implement JWT authentication with refresh token rotation`
- `[Core] Create core/ shared kernel — TenantContext, base ORM manager, and AuditLog model`

---

### [Users] Expose Users module REST API endpoints (DRF ViewSets and serializers)

**Summary**

With models, JWT, RBAC, and tenant-scoping in place, this issue wires the Users module's CRUD surface as DRF ViewSets per [03-ApiReference.md §3](03-ApiReference.md). Views are thin: parse → call service → serialise → return. Business logic lives in `apps/users/domain/services.py`, not in the view. Every write path emits an `AuditLog` row via `core/audit.py`.

**Work to do**

- Implement `apps/users/domain/services.py`:
  - `create_user(payload) -> User`
  - `update_user(user_id, payload) -> User`
  - `deactivate_user(user_id) -> None`
  - `create_membership(user_id, school_id, role_code) -> SchoolMembership`
  - `end_membership(membership_id) -> None`
  - Each service function calls `emit_audit_log(...)` from `core/audit.py`.
- Implement `apps/users/api/serializers.py` — `UserSerializer`, `SchoolMembershipSerializer`, `RoleSerializer` with field validation.
- Implement `apps/users/api/views.py` — `UserViewSet`, `SchoolMembershipViewSet` (ModelViewSet or explicit action views); apply `require_roles` decorators per [08-SecurityGuide.md §2](08-SecurityGuide.md) matrix.
- Register routes in `apps/users/api/urls.py`; include in `school_management_system/urls.py` under `/api/v1/users/`.
- Integration tests in `apps/users/tests/integration/test_users_api.py`:
  - `GET /api/v1/users/` — `registrar` can list, `student` is 403.
  - `POST /api/v1/users/` — `registrar` creates, duplicate email is 400.
  - `PATCH /api/v1/users/<id>/` — `registrar` updates, cross-tenant `id` is 404.
  - `DELETE /api/v1/users/<id>/` — soft-delete sets `is_active = False`.
  - Audit log row emitted on every write.

**Acceptance criteria**

- All Users endpoints in [03-ApiReference.md](03-ApiReference.md) return correct HTTP status codes.
- No business logic in views — all logic lives in `apps/users/domain/services.py`.
- Every write operation produces exactly one `AuditLog` row.
- Cross-tenant access returns 404 (not 403) to avoid confirming resource existence.
- RBAC matrix enforced: `registrar` CRUD, `teacher` read-own, `student` read-self.
- Integration tests pass.

**Blocked by**

- `[Users] Implement RBAC permission classes covering the 08-SecurityGuide.md matrix`
- `[Users] Implement tenant-scoping middleware and ORM manager`

---

## Area 4 — CI/CD Scaffold

---

### [CI/CD] Add GitHub Actions workflow: lint and type-check (ruff, black, mypy)

**Summary**

Per [01-Architecture.md §9](01-Architecture.md) (CI/CD stack: GitHub Actions + Docker) and the Phase 0 exit criteria in [11-Roadmap.md](11-Roadmap.md), CI must block merges on any failing lint or type gate. This workflow runs on every pull request and push to `main`. Per [04-DevelopmentGuide.md §4](04-DevelopmentGuide.md), Python style is enforced by Black (line length 100) and Ruff; type hints are required on the domain layer and enforced by `mypy --strict`.

**Work to do**

- Create `.github/workflows/lint.yml`:
  - Triggers: `pull_request`, `push` to `main`.
  - Job `lint`:
    - `actions/checkout@v4`
    - `astral-sh/setup-uv@v5` to install `uv`.
    - `uv sync --dev` to install all dependencies.
    - `uv run ruff check .` — fail on any lint error.
    - `uv run black --check --line-length 100 .` — fail on formatting differences.
    - `uv run mypy apps/*/domain/ core/` — fail on type errors in the domain and core layers.
  - Cache `uv`'s package cache using `actions/cache` keyed on `uv.lock`.
- Ensure `ruff`, `black`, and `mypy` are listed in `pyproject.toml` dev dependencies.
- Add `[tool.ruff]`, `[tool.black]`, and `[tool.mypy]` sections to `pyproject.toml` with the agreed settings.

**Acceptance criteria**

- Lint workflow runs on every pull request and passes on a clean codebase.
- A PR introducing an unused import is blocked by ruff.
- A PR with a formatting inconsistency is blocked by black.
- A PR adding a domain function without a return-type annotation is blocked by mypy.
- Workflow completes in under 3 minutes on a standard GitHub-hosted runner.

**Blocked by**

- `[DevOps] Add pre-commit hooks (ruff, black, mypy, bandit, gitleaks)` (for consistent tool config)

---

### [CI/CD] Add GitHub Actions workflow: tests (pytest with Postgres and Redis services)

**Summary**

Per [09-TestingGuide.md §2](09-TestingGuide.md), integration tests must run against a real PostgreSQL instance — never SQLite. Per the Phase 0 exit criteria in [11-Roadmap.md](11-Roadmap.md), CI must block merges on any failing test. This workflow spins up Postgres and Redis as service containers, runs `pytest`, and uploads a coverage report.

**Work to do**

- Create `.github/workflows/test.yml`:
  - Triggers: `pull_request`, `push` to `main`.
  - Job `test`:
    - Services: `postgres:16-alpine` (env: `POSTGRES_DB=sms_test`, `POSTGRES_USER=sms`, `POSTGRES_PASSWORD=sms`); `redis:7-alpine`.
    - `actions/checkout@v4`, `astral-sh/setup-uv@v5`, `uv sync --dev`.
    - Set environment variables: `DATABASE_URL`, `REDIS_URL`, `DJANGO_SECRET_KEY`, `JWT_SIGNING_KEY`.
    - `uv run python manage.py migrate --run-syncdb`.
    - `uv run pytest --cov=apps --cov=core --cov=sync --cov-report=xml --cov-fail-under=80`.
    - Upload `coverage.xml` as an artifact using `actions/upload-artifact`.
- Add `pytest`, `pytest-django`, `pytest-cov`, `factory-boy`, `pytest-asyncio` to `pyproject.toml` dev dependencies.
- Create `pytest.ini` (or `[tool.pytest.ini_options]` in `pyproject.toml`) pointing `DJANGO_SETTINGS_MODULE` to `school_management_system.settings`.

**Acceptance criteria**

- Test workflow runs on every pull request and passes with the current test suite.
- A PR introducing a failing test is blocked from merging.
- Coverage drops below 80% block the build (`--cov-fail-under=80`).
- Coverage report uploaded as a workflow artifact on every run.
- Services (Postgres, Redis) are healthy before tests start (use `options: --health-cmd`).

**Blocked by**

- `[DevOps] Switch development database from SQLite to PostgreSQL`
- `[CI/CD] Add GitHub Actions workflow: lint and type-check (ruff, black, mypy)`

---

### [CI/CD] Add GitHub Actions workflow: SAST and dependency scanning (Bandit, pip-audit, Trivy)

**Summary**

Per [08-SecurityGuide.md §9](08-SecurityGuide.md) and the Phase 0 exit criteria in [11-Roadmap.md](11-Roadmap.md), CI must run SAST (Bandit), dependency vulnerability scan (pip-audit), and container scan (Trivy) on every PR. HIGH and CRITICAL findings block the merge. This is the security gate referenced in [04-DevelopmentGuide.md §5](04-DevelopmentGuide.md) (PR requirements).

**Work to do**

- Create `.github/workflows/security.yml`:
  - Triggers: `pull_request`, `push` to `main`.
  - Job `sast`:
    - `uv run bandit -r apps/ core/ sync/ -ll` — fail on MEDIUM or higher findings (adjust threshold after initial baseline).
  - Job `dep-scan`:
    - `uv run pip-audit --requirement <(uv export --no-hashes)` — fail on any vulnerability with a fix available.
  - Job `container-scan` (runs after `[CI/CD] Add GitHub Actions workflow: Docker image build`):
    - Build image locally.
    - `aquasecurity/trivy-action` scanning the built image; fail on HIGH/CRITICAL CVEs.
- Add `bandit`, `pip-audit` to `pyproject.toml` dev dependencies.
- Document the severity thresholds and any known false-positive suppressions in a `bandit.yaml` config file.

**Acceptance criteria**

- Security workflow runs on every pull request.
- A PR introducing `subprocess.call(shell=True)` is flagged by Bandit and blocks the merge.
- A PR adding a dependency with a known HIGH CVE is flagged by pip-audit and blocks the merge.
- All three jobs (SAST, dep-scan, container-scan) must pass before a PR can merge.
- False-positive suppressions are documented and reviewed in `bandit.yaml` (not silently ignored inline).

**Blocked by**

- `[CI/CD] Add GitHub Actions workflow: lint and type-check (ruff, black, mypy)`

---

### [CI/CD] Add GitHub Actions workflow: Docker image build

**Summary**

Per [01-Architecture.md §9](01-Architecture.md) (Containerisation: Docker + Docker Compose) and the Phase 0 exit criteria in [11-Roadmap.md](11-Roadmap.md), every push to `main` must produce a verified, tagged Docker image. This ensures that every merge to main produces a deployable artefact and that the Dockerfile stays valid over time. The image build is also the prerequisite for the Trivy container scan in the security workflow.

**Work to do**

- Create `Dockerfile` (multi-stage build):
  - Stage `builder` — installs Python dependencies via `uv sync --no-dev` into a virtual env.
  - Stage `runtime` — `python:3.13-slim`, copies venv from builder, copies application code, sets `CMD ["gunicorn", "school_management_system.wsgi"]`.
  - No secrets or `.env` baked into the image.
- Create `.github/workflows/build.yml`:
  - Triggers: `push` to `main`; `pull_request` (build-only, no push).
  - Job `build`:
    - `docker/setup-buildx-action`.
    - `docker/login-action` (using `GITHUB_TOKEN`) on `main` branch only.
    - `docker/build-push-action`:
      - Tags: `ghcr.io/<org>/school-management-system:latest` and `:<sha>` on `main`.
      - On PRs: build only, no push (`push: false`).
      - Cache: `type=gha`.
- Update [10-DeploymentGuide.md](10-DeploymentGuide.md) with the image registry location and tagging scheme.

**Acceptance criteria**

- Docker image builds successfully on every push to `main` and on every PR (build-only for PRs).
- Built image starts with `docker run --env-file .env <image>` and serves the Django app.
- Image is pushed to GHCR only on `main` branch (not on PRs).
- Image build completes in under 5 minutes on a GitHub-hosted runner.
- A broken `Dockerfile` blocks the merge via a failing CI gate.

**Blocked by**

- `[DevOps] Add Docker Compose dev stack (Postgres, Redis, Celery, Django)`
- `[CI/CD] Add GitHub Actions workflow: tests (pytest with Postgres and Redis services)`

---

## Issue dependency graph

```
[DevOps] Docker Compose
    └── [DevOps] .env.example + settings.py
            └── [DevOps] SQLite → PostgreSQL
                    └── [Core] Scaffold apps/
                            └── [Core] core/ shared kernel
                                    ├── [Core] sync/ skeleton
                                    └── [Users] User + School models
                                            └── [Users] Membership + Role + Guardian
                                                    └── [Users] JWT auth
                                                            ├── [Users] RBAC permission classes
                                                            │       └── [Users] Users API endpoints
                                                            └── [Users] Tenant middleware
                                                                    └── [Users] Users API endpoints

[DevOps] Pre-commit hooks
    └── [CI/CD] Lint workflow
            ├── [CI/CD] Test workflow
            │       └── [CI/CD] Docker build workflow
            └── [CI/CD] SAST + dep scan workflow
                    └── [CI/CD] Docker build workflow (Trivy step)
```

---

## Summary table

| # | Issue | Module | Blocked by |
|---|-------|--------|-----------|
| 1 | Add Docker Compose dev stack | DevOps | — |
| 2 | Create .env.example and wire settings.py | DevOps | #1 |
| 3 | Switch dev database from SQLite to PostgreSQL | DevOps | #1, #2 |
| 4 | Add pre-commit hooks | DevOps | #2 |
| 5 | Scaffold apps/ bounded-context skeletons | Core | #3 |
| 6 | Create core/ shared kernel | Core | #5 |
| 7 | Create sync/ offline-sync skeleton | Core | #6 |
| 8 | Implement User and School models | Users | #6 |
| 9 | Implement SchoolMembership, Role, Guardian | Users | #8 |
| 10 | Implement JWT auth with refresh rotation | Users | #9 |
| 11 | Implement RBAC permission classes | Users | #6, #10 |
| 12 | Implement tenant-scoping middleware | Users | #6, #10 |
| 13 | Expose Users module REST API endpoints | Users | #11, #12 |
| 14 | GitHub Actions: lint + type-check | CI/CD | #4 |
| 15 | GitHub Actions: pytest + coverage | CI/CD | #3, #14 |
| 16 | GitHub Actions: SAST + dep scan | CI/CD | #14 |
| 17 | GitHub Actions: Docker image build | CI/CD | #1, #15 |
