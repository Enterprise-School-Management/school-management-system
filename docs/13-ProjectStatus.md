# 13 — Project Status

Current implementation status, locked decisions, open questions, and immediate next actions. This is the **living snapshot** of the project — update it whenever a phase boundary is crossed or a decision lands.

**Last updated:** 2026-04-17

**See also:** [11-Roadmap.md](11-Roadmap.md) · [02-ChangeLog.md](02-ChangeLog.md)

---

## Current Phase

**Phase 0 — Foundation** · Week 1–2 of the three-month delivery plan.

---

## Module Implementation Status

| Module | Design | Code | Tests | API | Notes |
|--------|--------|------|-------|-----|-------|
| Users | Drafted | Not started | Not started | Planned | Entities in [06-DomainModel.md](06-DomainModel.md). |
| Courses | Drafted | Not started | Not started | Planned | |
| Enrolments | Drafted | Not started | Not started | Planned | |
| Assessments | Drafted | Not started | Not started | Planned | Grade state machine documented. |
| Billing | Drafted | Not started | Not started | Planned | Paynow + ZIMRA flows designed. |
| Analytics | Drafted (Phase 1 rules only) | Not started | Not started | Planned | Predictive ML deferred. |
| Notifications | Drafted | Not started | Not started | Planned | In-app / email / SMS / push channels. |
| Sync (cross-cutting) | Drafted | Not started | Not started | Planned | Conflict semantics fixed for grades/payments. |

**Legend:** `Drafted` — design written · `Not started` / `Not yet implemented` — code pending · `Partial` / `Complete` — used once implementation begins.

---

## Infrastructure Status

| Component | Status | Notes |
|-----------|--------|-------|
| Python project via `uv` | **Complete** | [pyproject.toml](../../pyproject.toml) — Python ≥ 3.13, Django ≥ 6.0.4. |
| Django scaffold | **Complete** | [school_management_system/](../../school_management_system/) — default config + SQLite. |
| PostgreSQL (dev) | Not started | Planned for start of Phase 1a. |
| Redis | Not started | Planned for start of Phase 1a. |
| Celery workers | Not started | Planned for Phase 1a. |
| Docker Compose (dev) | Not started | Phase 0 deliverable. |
| Docker Compose (prod) | Not started | Phase 0–1 deliverable. |
| CI pipeline | Not started | Phase 0 deliverable — lint / SAST / dep scan / image build. |
| Pre-commit hooks | Not started | Phase 0. |
| Test harness (pytest + factories) | Not started | Lands with Phase 1a. |
| Next.js web client | Not started | Repo location decision: **TBD** (see Open Questions). |
| React Native mobile | Not started | Repo location decision: **TBD**. |

---

## Locked Decisions

Decisions below are treated as settled. Reversing any of them requires an ADR.

1. **Django 6 + DRF** as the backend framework.
2. **PostgreSQL 16** as the production database; SQLite is dev-only scaffold noise to be removed in Phase 1a.
3. **Row-level `school_id` multi-tenancy** — not schema-per-tenant, not database-per-tenant.
4. **Modular monolith first** — microservices only when forced by scale.
5. **DDD + Hexagonal architecture** for every domain module.
6. **JWT auth** — no cloud SSO dependency.
7. **Docker Compose** as the packaging target — no Kubernetes on school servers.
8. **React Native + Expo** for mobile — single codebase for Android + iOS.
9. **Next.js App Router + shadcn/ui + Tailwind** for web.
10. **`uv`** as the Python dependency / execution manager.
11. **AI is advisory** — no automated actions without a human actor (BR-AI-001).
12. **Grades and payments use explicit conflict flags** on sync; everything else is last-write-wins.

---

## Open Questions

| # | Question | Owner | Needed by |
|---|----------|-------|-----------|
| Q1 | Monorepo (backend + web + mobile) or separate repos? | `system_designer` | Phase 0 end |
| Q2 | Is TOTP MFA mandatory or role-opt-in for leadership? | `security_officer` | Phase 1a |
| Q3 | ZIMRA fiscal device vendor and procurement path | `finance_bursar` | Phase 1c |
| Q4 | SMS provider — single (e.g. Econet) or multi-gateway abstraction? | `system_designer` + `school_head` | Phase 1c |
| Q5 | Ministry reporting cadence and format | `registrar` | Phase 2 |
| Q6 | Cloud-backup target — per-school choice, or platform-provided default? | `school_head` | Phase 3 |
| Q7 | Does the first school deployment have a ZIMRA fiscal device already? | `finance_bursar` | Phase 3 |

---

## Immediate Next Actions

In rough order, scoped to the next two weeks:

1. Replace SQLite with PostgreSQL in `DATABASES`; introduce `.env` + `.env.example`.
2. Stand up `docker-compose.dev.yml` (postgres, redis, mailhog, minio).
3. Wire GitHub Actions CI: lint, Bandit, pip-audit, Trivy, Django `migrate --check`.
4. Draft ADR-0001 (tenancy model), ADR-0002 (JWT + refresh rotation), ADR-0003 (sync protocol).
5. Scaffold `apps/users/` with `School`, `User`, `SchoolMembership`, `Role` — end-to-end including tests and audit log.
6. Answer Q1 (repo structure) so frontend work can be planned.
7. Set up `.pre-commit-config.yaml`.

---

## Known Risks

See [11-Roadmap.md — Risks to Flag Early](11-Roadmap.md). Current live watches:

- **Paynow sandbox access** — needed before Phase 1c.
- **Low-end Android device parity** — needs a reference device pool before Phase 1b testing.
- **AI dataset cold-start** — mitigated by rule-based Phase 1; revisit at end of Phase 2.

---

## How to Keep This Document Accurate

1. At every phase boundary, update the **Current Phase** line and the **Module Implementation Status** table.
2. When a decision lands, move it from **Open Questions** to **Locked Decisions** and reference the ADR in [02-ChangeLog.md](02-ChangeLog.md).
3. When a risk materialises, move it to the "Known Risks" list with a mitigation plan.
4. Keep the page scannable — if it grows beyond ~250 lines, archive resolved entries into [02-ChangeLog.md](02-ChangeLog.md).
