# 09 — Testing Guide

Testing strategy, priority test areas, setup, and patterns for the School Management System.

**See also:** [01-Architecture.md](01-Architecture.md) · [12-BusinessRules.md](12-BusinessRules.md) · [04-DevelopmentGuide.md](04-DevelopmentGuide.md)

---

## 1. Test Pyramid

```
            ┌──────────────────────┐
            │   E2E / DR Drills    │   quarterly, few
            ├──────────────────────┤
            │  Load / Soak Tests   │   milestones
            ├──────────────────────┤
            │ Contract / Sync Tests│   per API / sync flow
            ├──────────────────────┤
            │   Integration Tests  │   per API endpoint
            ├──────────────────────┤
            │      Unit Tests      │   every domain service
            └──────────────────────┘
```

- **Unit** — fast, no I/O, cover every domain service and invariant. Target: `> 90%` coverage of the domain layer.
- **Integration** — spin up a real Postgres, exercise the DRF layer. Target: one happy path + one authorised-denied path per endpoint.
- **Contract** — verify the mobile/web client contracts against the API (pact-style or recorded fixtures).
- **Load** — scenarios in §5.
- **E2E / DR** — quarterly restore drills and golden-path browser/mobile runs.

---

## 2. Stack

| Layer | Tool | Notes |
|-------|------|-------|
| Runner | `pytest` + `pytest-django` | Standard. |
| Fixtures | `factory-boy` | One factory per aggregate root. |
| HTTP | DRF's `APIClient` | For integration tests. |
| DB | PostgreSQL (via Docker in CI) | **Never SQLite for integration tests** — behaviour drifts. |
| Async | `pytest-asyncio` | For ASGI and Celery eager-mode tests. |
| Coverage | `coverage.py` | Fails build below threshold (see §7). |
| Mocking | `unittest.mock` | For external adapters only (SMS, Paynow, email). |
| Load | `locust` | **Not yet implemented** — selected, pending setup. |
| Security | Bandit + Semgrep + Trivy | Gates in CI; see [08-SecurityGuide.md §9](08-SecurityGuide.md). |

---

## 3. Priority Test Areas

Ranked by risk. These get written first and kept green at all costs.

1. **Tenant isolation** — a request authenticated as School A must never touch School B data. One dedicated suite attempts every CRUD across tenants and asserts 404 / 403. See [07-DatabaseGuide.md §6](07-DatabaseGuide.md).
2. **Grade state machine** — every transition in [06-DomainModel.md](06-DomainModel.md); every rule `BR-GRD-*` in [12-BusinessRules.md](12-BusinessRules.md).
3. **Invoice / Payment / Receipt** — money is the highest-consequence data. Cover idempotency of `POST /payments`, fiscalisation retry, refund flow.
4. **Offline sync reconciliation** — client queue replays, conflict flagging on grades and payments, last-write-wins elsewhere.
5. **RBAC permission matrix** — one test per role × endpoint combination in [08-SecurityGuide.md §2](08-SecurityGuide.md). Generated from the matrix.
6. **Audit-log emission** — every write produces exactly one audit row in the same transaction.
7. **Report-card publication approval** — cannot publish without `school_head` signoff.
8. **Attendance offline submission** — mobile-submitted record with `sync_source = mobile` preserves device timestamp and reconciles server-side.
9. **Soft-delete visibility** — deleted rows are excluded from default queries and from audit searches as appropriate.
10. **Notification delivery paths** — templates render; channel routing respects preferences.

---

## 4. Conventions

### Test layout

```
<module>/
  tests/
    unit/
      test_<service>.py
    integration/
      test_<endpoint>.py
    fixtures/
      factories.py
```

### Naming

- `test_<what>_<condition>_<expected>` — e.g. `test_grade_approve_by_teacher_returns_403`.
- One behaviour per test. Arrange-act-assert blocks separated by a blank line.

### Factories

- One factory per aggregate root.
- `SchoolFactory` produces a tenant and wires `school_id` into sub-factories.
- Never use Django fixtures (`.json`) for tests — factories only.

### Database

- Each test runs in a transaction that rolls back at teardown.
- Shared tenant created per test module; avoid accidental cross-test pollution.

### Time

- Freeze time with `freezegun` for any test asserting state transitions, due-date sweeps, or token expiry.

---

## 5. Load Scenarios

Mirrors [docs/spec.md §11.2](../../docs/spec.md):

| Scenario | Target | Metric |
|----------|--------|--------|
| Enrolment spike at term start | 50 concurrent registrars creating 500 students in 10 min | p95 < 400 ms on `POST /students` |
| Exam-window quiz submissions | 300 students submitting within a 5-min window | 0 dropped submissions; p95 < 800 ms |
| Bulk lesson-content upload | 100 lessons × 10 MB each, LAN | Throughput > 50 Mbps on school LAN |
| Concurrent mobile sync | 200 devices pushing 20 ops each | All ops acknowledged within 2 min |

Load runs land in Phase 3 (see [11-Roadmap.md](11-Roadmap.md)).

---

## 6. Disaster Recovery Drills

Every quarter:

1. Restore the latest backup to a clean Postgres instance.
2. Assert: audit-log continuity, latest invoice totals equal pre-restore, no tenant leak introduced.
3. Time the restore — record RTO and compare against the 2-hour target.
4. File a note in [02-ChangeLog.md](02-ChangeLog.md) under "Operations".

---

## 7. Coverage & Quality Gates

| Gate | Threshold |
|------|-----------|
| Domain-layer coverage | ≥ 90% |
| Overall coverage | ≥ 80% |
| No test skipped in CI without a linked ticket reference | Enforced via lint rule. |
| No test depending on wall-clock timing | `freezegun` mandatory. |
| No test hitting external services | Mocks for SMS, Paynow, email. |

---

## 8. AI-Generated Code Policy

Per [docs/spec.md §15.2](../../docs/spec.md) and the project's multi-agent rules:

- **Every AI-generated module ships with tests.** No exceptions.
- **Human review mandatory** before merge.
- Tests must exercise the business rules referenced in the change, not just happy paths.

---

## 9. Running Tests Locally

> Commands apply once the test harness is wired up in Phase 1. Until then, only the default Django test runner is available.

```bash
# All tests
uv run pytest

# One module
uv run pytest enrolments/

# With coverage
uv run pytest --cov --cov-report=term-missing

# Only integration
uv run pytest -m integration
```

Markers:

- `@pytest.mark.integration` — needs live DB.
- `@pytest.mark.sync` — exercises sync adapter.
- `@pytest.mark.slow` — skipped by default; run in nightly CI.

---

## 10. Status

Test scaffolding is **Not yet implemented**. Landing alongside the first domain module in Phase 1 per [11-Roadmap.md](11-Roadmap.md).
