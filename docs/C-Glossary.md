# Glossary

Shared vocabulary used across the School Management System documentation and codebase. Every term used in more than one document should be defined here first.

**See also:** [A-INDEX.md](A-INDEX.md) · [01-Architecture.md](01-Architecture.md) · [06-DomainModel.md](06-DomainModel.md)

---

## Domain Terms — Academic

| Term | Definition |
|------|------------|
| **Academic Year** | Calendar year in which a cohort progresses through one grade level. Scoped per school. |
| **Term** | Subdivision of an academic year (typically three terms in the Zimbabwean system). Drives fee cycles, report-card publication, and promotion decisions. |
| **Class** | A concrete group of students taught together for a given course during a term (e.g. "Form 2B Mathematics, Term 1 2026"). Distinct from a Course. |
| **Course** | Reusable subject definition (e.g. "Mathematics — Form 2"). A Course is instantiated into one or more Classes each term. |
| **Enrolment** | The link between a Student and a Class (or School). Carries status (active, withdrawn, graduated), date, and fee obligations. |
| **Timetable** | The weekly schedule of Classes with rooms and teachers assigned. |
| **Gradebook** | Teacher-facing ledger of every Grade awarded for a Class. Publishes to the Report Card. |
| **Report Card** | Student-facing summary of grades and comments for a Term. Requires publication approval from school leadership. |
| **Assessment** | An evaluated piece of work: Assignment, Quiz, or Exam. Belongs to a Class; produces Grades. |
| **Attendance** | Daily presence record per Student per Class. Offline-first; syncs from mobile. |
| **Promotion** | End-of-year decision moving a student to the next grade level. Governed by rules in [12-BusinessRules.md](12-BusinessRules.md). |
| **Cohort** | Set of students sharing a grade level and academic year. |

## Domain Terms — Finance

| Term | Definition |
|------|------------|
| **Fee Head** | A category of chargeable fee (tuition, boarding, transport, uniform). Each Fee Head has its own amount schedule. |
| **Invoice** | A bill raised against a Student or Guardian for one or more Fee Heads in a given Term. |
| **Receipt** | Confirmation of payment against an Invoice. Links to a Payment record and carries a ZIMRA fiscal reference when fiscalised. |
| **Payment** | Money-movement record: cash, EcoCash, OneMoney, bank transfer, or Paynow callback. |
| **Reconciliation** | Process of matching Payments to Invoices and bank statements, resolving discrepancies. |
| **ZIMRA Fiscalisation** | Legal requirement in Zimbabwe for certain receipts to be issued through a fiscal device and reported to ZIMRA. See [12-BusinessRules.md](12-BusinessRules.md). |
| **Scholarship** | Recorded reduction or waiver of a Fee Head for a specific Student. Requires approval. |
| **Paynow** | Zimbabwean payment aggregator used for EcoCash, OneMoney, and bank transfers. See [developers.paynow.co.zw](https://developers.paynow.co.zw/docs/initiate_mobile_transaction.htm). |

## Workflow Terms

| Term | Definition |
|------|------------|
| **Offline-First** | Design principle: every feature must work without internet access. The local school server is authoritative; internet is optional. |
| **Sync Queue** | Device-local queue (WatermelonDB on mobile, service-worker store on web) of mutations awaiting upload to the school server. |
| **Conflict Flag** | Marker raised when two writes to the same record disagree and automatic resolution is unsafe (used for grade and payment changes). Resolved by a human. |
| **Last-Write-Wins** | Default conflict-resolution strategy for low-risk data (profile edits, attendance). |
| **Plugin Module** | A domain module that a school activates or deactivates. Drives tiered pricing without code changes. |
| **Cloud Backup** | Opportunistic sync of the local school server to a remote backup destination when internet is available. Encrypted. |
| **Superadmin Dashboard** | Cross-school view used by the platform operator to monitor all deployed schools. Receives aggregates only when connectivity allows. |
| **AI-Advisory** | Principle that AI output is a recommendation; humans make the final decision. Applies to grading, risk flags, and discipline. |
| **Approval Workflow** | Multi-step flow where one role submits and another approves (e.g. grade override, report-card publication, scholarship). |

## Technical Terms

| Term | Definition |
|------|------------|
| **Modular Monolith** | One deployable Django app composed of independent domain modules with clean internal boundaries. See [01-Architecture.md](01-Architecture.md). |
| **DDD** | Domain-Driven Design — organise code around business domains, not technical layers. |
| **Hexagonal Architecture** | Ports-and-adapters pattern: the domain core depends on nothing; infrastructure plugs in at the edges. |
| **Bounded Context** | A consistent model for one domain. Each module in §5 of [docs/spec.md](../../docs/spec.md) is a bounded context. |
| **Aggregate Root** | The single entity that governs writes to a cluster of related objects (e.g. Invoice is the root of Invoice + InvoiceLines). |
| **Multi-Tenancy** | One installation serves multiple schools. Implemented as row-level scoping on `school_id`. |
| **Row-Level Tenancy** | Every tenant-scoped table carries a `school_id` FK; queries filter on it. No shared rows between schools. |
| **Soft Delete** | Setting a `deleted_at` timestamp instead of removing a row. Required for audit and recovery. |
| **Audit Log** | Append-only record of every write: who, when, what changed, before/after values. |
| **WatermelonDB** | Local SQLite-backed database on React Native mobile apps. Holds offline cache and sync queue. |
| **pgvector** | PostgreSQL extension for vector similarity search. Used for AI/RAG features. |
| **RAG** | Retrieval-Augmented Generation. An LLM is prompted with relevant chunks fetched from a vector store. See [docs/spec.md §6.3](../../docs/spec.md). |
| **ASGI** | Async Server Gateway Interface. Enables Django to serve real-time workloads. |
| **Celery** | Task queue used for async work (email, SMS, grading jobs, analytics, backups). |
| **PWA** | Progressive Web App. The Next.js web client uses service workers to cache critical content for offline use. |
| **JWT** | JSON Web Token, used for stateless authentication of API clients. |
| **RBAC** | Role-Based Access Control. Each role (see [.claude/agents/](../agents/)) has a defined permission set. |
| **SAST** | Static Application Security Testing. Runs in CI on every change. |
| **DR Drill** | Disaster-Recovery Drill. Scheduled test of backup restore and recovery procedures. |

## Zimbabwe Context

| Term | Definition |
|------|------------|
| **ATS School** | Authorised Tuition School — a private school category recognised by the Zimbabwe Ministry of Primary and Secondary Education. Primary target market. |
| **ZESA** | Zimbabwe Electricity Supply Authority. Frequent power outages are a design constraint (UPS, offline-capable workflows). |
| **EcoCash / OneMoney** | Mobile-money wallets. Common payment rails. |
| **Econet / NetOne** | Zimbabwean mobile networks used as SMS gateways. |
| **ZIMRA** | Zimbabwe Revenue Authority. Issues the fiscal-device compliance requirements for receipts. |
| **DPA** | Zimbabwe Cyber Security and Data Protection Act. Governs handling of student and staff personal data. |
