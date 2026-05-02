# 01 — Architecture

Layer structure, module responsibilities, dependency rules, and technology choices for the School Management System. Mirrors [docs/spec.md §2–§5](../../docs/spec.md); that spec is the source of truth when these two documents disagree.

**See also:** [C-Glossary.md](C-Glossary.md) · [06-DomainModel.md](06-DomainModel.md) · [07-DatabaseGuide.md](07-DatabaseGuide.md) · [docs/spec.md](../../docs/spec.md)

---

## 1. Core Principles

Mirrors [docs/spec.md §1 — Core Principles](../../docs/spec.md).

- **Offline-first everywhere** — every feature must work without internet access.
- **Multi-tenant from day one** — one installation serves multiple schools with full data isolation.
- **Mobile is first-class** — Android and iOS delivered alongside the web app, not deferred.
- **AI-advisory, not AI-final** — AI assists grading and discipline decisions; humans decide.
- **Plugin-based modules** — schools activate only what they need; tiered pricing without code changes.
- **Zimbabwe-context fit** — designed for ZESA outages, LAN-only networks, low-end Android devices, and local payment and SMS providers.

---

## 2. Layer Diagram

```
+---------------------------+     +---------------------------+
|   Web Client (Next.js)    |     | Mobile Client (RN + Expo) |
|   PWA, service workers    |     | WatermelonDB + sync queue |
+------------+--------------+     +-------------+-------------+
             |                                  |
             |  HTTPS over LAN                  |  HTTPS over LAN
             v                                  v
+-----------------------------------------------------------+
|                   API Gateway (DRF)                       |
|           JWT auth · tenant scoping · versioning          |
+-----------------------------------------------------------+
|                    Domain Modules                         |
|   Users · Courses · Enrolments · Assessments ·            |
|   Billing · Analytics · Notifications                     |
|            (DDD + Hexagonal per module)                   |
+-----------------------------------------------------------+
|        Async Workers (Celery)   |    Event Bus (Redis)    |
|  email · SMS · grading · AI · backups · sync reconcilers  |
+-----------------------------------------------------------+
|   Data Layer                                              |
|   PostgreSQL (tenant-scoped) · Redis (cache + broker) ·   |
|   pgvector (AI/RAG) · Local filesystem (files)            |
+-----------------------------------------------------------+
|                  School Server (Docker)                   |
|   Local LAN · UPS-backed · cloud backup when connected    |
+-----------------------------------------------------------+
```

---

## 3. Pattern: Modular Monolith First

Mirrors [docs/spec.md §2.1](../../docs/spec.md).

The system starts as a **modular monolith**, not microservices. Microservices are introduced later only for high-change or high-scale areas such as video processing, search, AI inference, and notifications (see §10).

**Why this works:**
- Faster to ship than full microservices.
- Easier testing and maintenance.
- Clean domain boundaries make later extraction to services straightforward.
- Django ASGI support enables real-time without a full service split.

---

## 4. Architectural Patterns

Mirrors [docs/spec.md §2.2](../../docs/spec.md).

| Pattern | Application |
|---------|-------------|
| **DDD + Hexagonal Architecture** | Clean domain boundaries; core domains listed in §5. |
| **MVC** | Web app and admin UI. |
| **Event-driven** | Async work: email, grading jobs, analytics pipelines, recommendations, audit logs. |
| **Plugin Architecture** | Modules activated or deactivated per school for tiered features and pricing. |
| **Multi-tenancy** | Row-based `school_id` scoping with full data isolation per school. |

---

## 5. Domain Modules

Mirrors [docs/spec.md §5](../../docs/spec.md). Each module is a self-contained bounded context with its own models, services, repositories, API surface, and event emitters. Activation is per-school via the Plugin Architecture.

| Module | Responsibilities |
|--------|------------------|
| **Schools** | School registration, configuration, timezone, active-module list, billing tier |
| **Users** | Authentication, profiles, roles, permissions, school memberships |
| **Courses** | Course catalogue, lessons, content, question banks |
| **Enrolments** | Student enrolment, class assignment, timetables |
| **Assessments** | Assignments, quizzes, grading, gradebook, report cards |
| **Billing** | Fee management, invoicing, payments, ZIMRA compliance |
| **Analytics** | Engagement scoring, risk flags, recommendations, dashboards |
| **Notifications** | In-app, email, SMS, push; delivered via Celery queue |

> This list is the canonical module set. It must match the module headings in [06-DomainModel.md](06-DomainModel.md) and the endpoint groups in [03-ApiReference.md](03-ApiReference.md).

---

## 6. Dependency Rules

Extension of the DDD + Hexagonal pattern. Not in the spec, but consistent with it and binding on the codebase.

1. **Domain core → nothing.** Domain services and entities do not import Django, DRF, Celery, or any infrastructure.
2. **Infrastructure → domain.** Repositories, API views, Celery tasks import and call domain services — never the other way.
3. **Cross-module → via events only.** Module A does not directly call Module B's internal services. It emits an event; Module B subscribes.
4. **Shared kernel is small.** Only truly universal concepts (`SchoolId`, audit-log helpers, `Money`) live in a shared package. Everything else stays inside its module.
5. **One-way imports between modules.** If A needs B's read model, A imports B's public interface only (its events and thin read adapters), never B's internals.

Violations of rules 1–3 must be rejected in code review.

---

## 7. Multi-Tenancy

Mirrors [docs/spec.md §2.3](../../docs/spec.md).

- All data is scoped by `school_id` at the row level.
- A **superadmin dashboard** monitors all schools from a single installation.
- Cross-school aggregates sync to the superadmin when internet is available.

### Enforcement

- A request-scoped `TenantContext` resolves `school_id` from the authenticated user's claims before any query runs.
- Base querysets are wrapped in a tenant-aware manager that injects `school_id` filters automatically.
- Cross-tenant queries (superadmin dashboard) go through a separate explicit API; they never leak into regular endpoints.
- Data isolation is verified by integration tests; see [09-TestingGuide.md](09-TestingGuide.md).

### Superadmin Authentication

Superadmin access is **a role on the same auth system** — not a separate login page or a separate Django site. See [ADR-0001](adr/0001-superadmin-authentication.md) for the full decision record.

Summary:

- The platform operator is a `User` with a `SchoolMembership` where `role = system_designer` (or `security_officer`) and `is_superadmin = True`. The membership's `school_id` references a reserved **platform tenant** (`short_code = __platform__`) created on first install.
- Login goes through the same `POST /api/v1/auth/login` endpoint used by all other roles.
- The JWT issued for a superadmin includes `"superadmin": true` in its claims.
- A dedicated `IsSuperadmin` DRF permission class is applied at the router level to every `/api/v1/super/*` endpoint. Requests without the claim receive `403 Forbidden`.
- `is_superadmin` is **never writable via the public API** — it can only be set through Django admin or a migration fixture.

---

## 8. Offline-First Strategy

Mirrors [docs/spec.md §4](../../docs/spec.md). Offline-first is the **primary design assumption**, not a fallback mode. The system runs on a local school server. Teachers and students connect via the school's LAN or WiFi. No internet is required for any day-to-day operation.

### 8.1 Platform Targets

| Platform | Technology | Access Method | Offline Capability |
|----------|------------|---------------|--------------------|
| Web App | Next.js | Browser on school LAN | Service workers (PWA) |
| Android | React Native + Expo | Native app on school WiFi | WatermelonDB + sync queue |
| iOS | React Native + Expo | Native app on school WiFi | WatermelonDB + sync queue |
| Desktop | Web app via browser | Browser on school LAN | Full local server access |

### 8.2 Sync Strategy

- **Mobile devices** queue offline actions locally in WatermelonDB and sync to the school server when reconnected to the LAN or internet.
- **School server** syncs to cloud backup when internet is available — automated, scheduled, encrypted.
- **Software and security updates** are pulled from the cloud update server when connectivity allows.
- **Cross-school aggregates** sync to the superadmin dashboard when connectivity allows.

**WatermelonDB sync endpoint (DRF side):** The mobile clients push mutations to and pull server changes from `POST /api/v1/sync/push` and `POST /api/v1/sync/pull`. A third endpoint, `POST /api/v1/sync/resolve`, lets a client submit a human-resolved conflict. These are owned by the **Enrolments** module team in Phase 1 (attendance and student data are the first offline-sync targets) and extended by other modules as they land. See [03-ApiReference.md §10](03-ApiReference.md) for the full sync endpoint specification.

### 8.3 Conflict Resolution

| Data Type | Strategy |
|-----------|----------|
| Most data | Last-write-wins |
| Grade changes | Explicit conflict flagging |
| Payment records | Explicit conflict flagging |

**Who sees the flag and how they resolve it:**

- When a grade or payment conflict is detected on the server, the record's status transitions to `conflict_flagged` (see state machines in [06-DomainModel.md](06-DomainModel.md)).
- The `examinations_officer` role sees grade conflicts; the `finance_bursar` role sees payment conflicts. Both are surfaced in an in-app notification and a dedicated "Conflicts" view in the dashboard.
- The resolver opens the conflict detail screen, which shows the two competing versions side-by-side (device timestamp, user, value). They select one version or enter a corrected value and submit via `POST /api/v1/sync/resolve`. The server transitions the record to `approved` (grades) or `confirmed` (payments) and logs the resolution to the audit log.
- Unresolved conflicts older than 7 days escalate to the `school_head` role via a daily digest notification.

See [C-Glossary.md](C-Glossary.md) for **Conflict Flag** and **Sync Queue** definitions.

---

## 9. Technology Stack

Mirrors [docs/spec.md §3](../../docs/spec.md).

### 9.1 Summary

| Layer | Technology | Notes |
|-------|------------|-------|
| **Web Frontend** | Next.js (App Router) | PWA with service workers for offline caching |
| **Mobile** | React Native + Expo | Android and iOS; WatermelonDB for offline storage |
| **Backend** | Django + Django REST Framework | Modular monolith, ASGI-ready |
| **Database** | PostgreSQL | Local, runs on school server |
| **Cache** | Redis (self-hosted) | Also used as message broker. Separated by DB index (DB 0 = cache, DB 1 = broker) in Phase 1. See [ADR-0002](adr/0002-redis-cache-and-broker.md). |
| **Queue** | Celery + Redis | Async jobs: email, grading, analytics, backups. Celery workers run as a **separate service** in the same `docker-compose.yml` as the API (`celery_worker` service, same image). |
| **Search** | Redis first; OpenSearch/Elasticsearch if performance demands it | MongoDB only if a real document use case appears |
| **Auth** | Django Auth + JWT | No cloud SSO required for local deployment |
| **File Storage** | Local filesystem | Cloud backup when connectivity available |
| **Containerisation** | Docker + Docker Compose | Consistent deployment on any school server hardware |
| **CI/CD** | GitHub Actions + Docker | Lint, SAST, dependency scan, image build, deploy |
| **Vector DB** | pgvector first; Pinecone or Weaviate if scale demands | For AI/RAG features. Move to a dedicated vector store when **any** of: (a) vector index size exceeds 10 M embeddings, (b) pgvector query latency p95 > 200 ms, or (c) AI inference module is extracted to a microservice. |
| **UI Components** | shadcn/ui | Component library |
| **Styling** | Tailwind CSS | Default with shadcn |
| **Mobile Offline** | WatermelonDB | Local storage and sync queue on device |

### 9.2 Components Deferred (Not Required for Local Deployment)

Introduced when the system grows to a hosted multi-school cloud platform:

- Kubernetes
- S3 / cloud object storage
- Keycloak / Auth0
- ELK stack, Prometheus, Grafana
- Cloud deployment (AWS, Azure, GCP)

---

## 10. Future Microservices Candidates

Mirrors [docs/spec.md §2.4](../../docs/spec.md). Extract to separate services only when scale demands it:

- Video processing
- Full-text search
- AI inference
- Notification delivery

The clean module boundaries in §5 make each extraction a code-movement exercise, not a redesign.

### Interface stubs — what to define now

No microservice needs to be built now. However, to avoid a painful extraction later, two interface boundaries should be defined before the relevant module ships:

| Module | Stub to define now | Reason |
|--------|--------------------|--------|
| **Notifications** | An abstract `NotificationBackend` port (send method signature, payload contract) | Notification delivery is the highest-probability extraction candidate. Abstracting the delivery backend today means swapping Celery in-process delivery for a separate notification service later requires only a new adapter, not refactoring every call site. |
| **AI / Analytics** | An abstract `InferenceBackend` port (predict method signature, model-name contract) | AI inference load is unpredictable. An interface means the same domain code works whether the model runs in-process (Phase 1) or behind an HTTP call to a separate service (later). |

Full-text search and video processing have no implementation in Phase 1, so no stub is needed yet. Define their interfaces when the feature design starts.

---

## 11. Architectural Decision Records (ADRs)

ADRs live under [`docs/adr/`](adr/README.md). Each ADR captures context, decision, consequences, and status. Reference ADRs by number from [02-ChangeLog.md](02-ChangeLog.md) when a decision lands.

| ADR | Title | Status |
|-----|-------|--------|
| [ADR-0001](adr/0001-superadmin-authentication.md) | Superadmin Authentication Strategy | Accepted |
| [ADR-0002](adr/0002-redis-cache-and-broker.md) | Redis Dual-Use: Cache and Message Broker | Accepted |

To record a new architectural decision, copy [`docs/adr/0000-template.md`](adr/0000-template.md) to a new numbered file, fill in all sections, and open a PR for review.
