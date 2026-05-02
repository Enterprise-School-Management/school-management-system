# School Management System — Technical Specification

**Version:** 2.0
**Target Market:** Zimbabwean ATS Schools
**Primary Constraint:** Offline-First
**Last Updated:** 2026-04-09

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Stack](#3-stack)
4. [Offline-First Strategy](#4-offline-first-strategy)
5. [Domain Modules](#5-domain-modules)
6. [AI Analytics](#6-ai-analytics)
7. [Integrations](#7-integrations)
8. [Security & Compliance](#8-security--compliance)
9. [Frontend Guidelines](#9-frontend-guidelines)
10. [Mobile Guidelines](#10-mobile-guidelines)
11. [Testing Strategy](#11-testing-strategy)
12. [Three-Month Delivery Plan](#12-three-month-delivery-plan)
13. [MVP Scope](#13-mvp-scope)
14. [Deferred Features](#14-deferred-features)
15. [Multi-Agent Development Workflow](#15-multi-agent-development-workflow)
16. [Post-Launch Operations](#16-post-launch-operations)

---

## 1. Overview

A full-featured, offline-first school management system built for Zimbabwean ATS schools. The system runs on a **local school server** with no cloud dependency at runtime. Internet connectivity is treated as a bonus, not a baseline.

### Core Principles

- **Offline-first everywhere** — every feature must work without internet access.
- **Multi-tenant from day one** — one installation serves multiple schools with full data isolation.
- **Mobile is first-class** — Android and iOS delivered alongside the web app, not deferred.
- **AI-advisory, not AI-final** — AI assists grading and discipline decisions; humans decide.
- **Plugin-based modules** — schools activate only what they need; tiered pricing without code changes.
- **Zimbabwe-context fit** — designed for ZESA outages, LAN-only networks, low-end Android devices, and local payment and SMS providers.

---

## 2. Architecture

### 2.1 Pattern: Modular Monolith First

The system starts as a **modular monolith**, not microservices. Microservices are introduced later only for high-change or high-scale areas such as video processing, search, AI inference, and notifications.

**Why this works:**
- Faster to ship than full microservices.
- Easier testing and maintenance.
- Clean domain boundaries make later extraction to services straightforward.
- Django ASGI support enables real-time without a full service split.

### 2.2 Architectural Patterns

| Pattern | Application |
|---|---|
| **DDD + Hexagonal Architecture** | Clean domain boundaries; core domains listed in §5 |
| **MVC** | Web app and admin UI |
| **Event-driven** | Async work: email, grading jobs, analytics pipelines, recommendations, audit logs |
| **Plugin Architecture** | Modules activated or deactivated per school for tiered features and pricing |
| **Multi-tenancy** | Row-based `school_id` scoping with full data isolation per school |

### 2.3 Multi-Tenancy

- All data is scoped by `school_id` at the row level.
- A **superadmin dashboard** monitors all schools from a single installation.
- Cross-school aggregates sync to the superadmin when internet is available.

### 2.4 Future Microservices Candidates

Extract to services only when scale demands it:
- Video processing
- Full-text search
- AI inference
- Notification delivery

---

## 3. Stack

### 3.1 Summary

| Layer | Technology | Notes |
|---|---|---|
| **Web Frontend** | Next.js (App Router) | PWA with service workers for offline caching |
| **Mobile** | React Native + Expo | Android and iOS; WatermelonDB for offline storage |
| **Backend** | Django + Django REST Framework | Modular monolith, ASGI-ready |
| **Database** | PostgreSQL | Local, runs on school server |
| **Cache** | Redis (self-hosted) | Also used as message broker |
| **Queue** | Celery + Redis | Async jobs: email, grading, analytics, backups |
| **Search** | Redis first; OpenSearch/Elasticsearch if performance demands it | MongoDB only if a real document use case appears |
| **Auth** | Django Auth + JWT | No cloud SSO required for local deployment |
| **File Storage** | Local filesystem | Cloud backup when connectivity available |
| **Containerisation** | Docker + Docker Compose | Consistent deployment on any school server hardware |
| **CI/CD** | GitHub Actions + Docker | Lint, SAST, dependency scan, image build, deploy |
| **Vector DB** | pgvector first; Pinecone or Weaviate if scale demands | For AI/RAG features |
| **UI Components** | shadcn/ui | Component library |
| **Styling** | Tailwind CSS | Default with shadcn |
| **Mobile Offline** | WatermelonDB | Local storage and sync queue on device |

### 3.2 Components Deferred (Not Required for Local Deployment)

These are introduced when the system grows to a hosted multi-school cloud platform:

- Kubernetes
- S3 / cloud object storage
- Keycloak / Auth0
- ELK stack, Prometheus, Grafana
- Cloud deployment (AWS, Azure, GCP)

---

## 4. Offline-First Strategy

Offline-first is the **primary design assumption**, not a fallback mode. The system runs on a local school server. Teachers and students connect via the school's LAN or WiFi. No internet is required for any day-to-day operation.

### 4.1 Platform Targets

| Platform | Technology | Access Method | Offline Capability |
|---|---|---|---|
| Web App | Next.js | Browser on school LAN | Service workers (PWA) |
| Android | React Native + Expo | Native app on school WiFi | WatermelonDB + sync queue |
| iOS | React Native + Expo | Native app on school WiFi | WatermelonDB + sync queue |
| Desktop | Web app via browser | Browser on school LAN | Full local server access |

### 4.2 Sync Strategy

- **Mobile devices** queue offline actions locally in WatermelonDB and sync to the school server when reconnected to the LAN or internet.
- **School server** syncs to cloud backup when internet is available — automated, scheduled, encrypted.
- **Software and security updates** are pulled from the cloud update server when connectivity allows.
- **Cross-school aggregates** sync to the superadmin dashboard when connectivity allows.

### 4.3 Conflict Resolution

| Data Type | Strategy |
|---|---|
| Most data | Last-write-wins |
| Grade changes | Explicit conflict flagging |
| Payment records | Explicit conflict flagging |

---

## 5. Domain Modules

The system is organised around the following core domains using DDD + Hexagonal Architecture:

| Domain | Responsibilities |
|---|---|
| **Schools** | School entity lifecycle, tenant registration, module activation flags, TenantContext resolution |
| **Users** | Authentication, profiles, roles, permissions |
| **Courses** | Course catalogue, lessons, content, question banks |
| **Enrolments** | Student enrolment, class assignment, timetables |
| **Assessments** | Assignments, quizzes, grading, gradebook, report cards |
| **Billing** | Fee management, invoicing, payments, ZIMRA compliance |
| **Analytics** | Engagement scoring, risk flags, recommendations, dashboards |
| **Notifications** | In-app, email, SMS, push; delivered via Celery queue |

Each domain is a self-contained module with:
- Its own models, services, repositories, and API endpoints
- Activation toggle per school (plugin architecture)
- Event emitters for async cross-domain communication

---

## 6. AI Analytics

AI features are implemented in phases. AI is **advisory only** — it never makes final decisions on grading or discipline.

### 6.1 Phase 1 — Dashboards & Rules (MVP)
- Engagement dashboards
- Engagement scoring
- At-risk student flags
- Recommendation rules

### 6.2 Phase 2 — Prediction & Personalisation
- Dropout risk classification models
- Learning-path personalisation
- NLP grading assistance
- Sentiment analysis

### 6.3 Phase 3 — RAG Tutor
- Retrieval-Augmented Generation tutor over school content
- AI assistant for policies, FAQs, assessment rubrics

### 6.4 AI Stack

| Component | Technology |
|---|---|
| ML frameworks | PyTorch, TensorFlow, scikit-learn |
| Feature engineering | Pandas + dbt |
| Vector DB | pgvector (first); Pinecone or Weaviate at scale |
| LLM retrieval | RAG over course content, policies, FAQs, rubrics |

### 6.5 Model Types

| Use Case | Model Type |
|---|---|
| Dropout risk | Classification |
| Content recommendation | Ranking / recommendation |
| Rubric alignment | NLP |
| Feedback drafts | NLP generation |
| Sentiment tagging | NLP classification |

---

## 7. Integrations

All integrations are designed for the Zimbabwe context. Internet-dependent integrations degrade gracefully when offline.

### 7.1 Video
- **Online:** Zoom or Google Meet
- **Offline:** Local recorded content fallback

### 7.2 Email
- **Online:** SendGrid or Postmark
- **Offline:** Local Celery queue; delivered when connectivity restores

### 7.3 File Storage
- **Primary:** Local filesystem on school server
- **Backup:** Automated cloud backup when connectivity available

### 7.4 Push Notifications & SMS
- **Push:** Firebase Cloud Messaging
- **SMS:** Local Zimbabwe gateways — Econet, NetOne

### 7.5 Payments
- **Primary:** EcoCash, OneMoney, bank transfer
- **Gateway:** [Paynow Zimbabwe](https://developers.paynow.co.zw/docs/initiate_mobile_transaction.htm)
- **Secondary:** Stripe or PayPal if international expansion occurs

### 7.6 Identity
- **Primary:** Django Auth + JWT
- **Optional:** Microsoft Entra or Google Workspace — only if a specific school requires it

### 7.7 Compliance
- ZIMRA Fiscal device compliance for billing
- Zimbabwe Ministry of Primary and Secondary Education reporting requirements
- GDPR, FERPA, COPPA — applicable only if system expands internationally
- Zimbabwe Cyber Security and Data Protection Act (DPA)
- Further compliance integrations to be specified at a later date

---

## 8. Security & Compliance

### 8.1 Security Requirements

- TLS everywhere on the local network
- Row-level audit logs for all write operations
- Least-privilege role-based access control (RBAC)
- Dependency and container scanning in CI before every deploy
- Automated backups with point-in-time recovery — local first, cloud when connectivity available

### 8.2 Access Control

- Plugin architecture enforces module-level access per school
- Multi-tenant row-based scoping enforces school-level data isolation
- RBAC enforces least-privilege at the user level within each school

### 8.3 CI Security Gates

Every deployment pipeline must include:
- Lint
- SAST (static application security testing)
- Dependency vulnerability scan
- Container image scan
- Human review for AI-generated code

---

## 9. Frontend Guidelines

### 9.1 Web App (Next.js)

- **Router:** App Router (`/app` folder) — not the legacy Pages Router
- **Component Library:** [shadcn/ui](https://ui.shadcn.com/)
- **Styling:** Tailwind CSS (default with shadcn)
- **Offline:** PWA service workers for offline caching of critical content
- **Accessibility:** WCAG 2.1 AA — design for low-bandwidth and low-spec devices

### 9.2 UX Principles

- Simple navigation with low click count to reach core actions
- Mobile-first — optimise for low-end Android devices
- Low-bandwidth asset optimisation
- Offline-capable critical paths (attendance, lesson access, assignment submission)

### 9.3 Content Features

- Version lessons with draft and published states
- Reusable question banks across assessments
- Gamification: badges, streaks, progress bars
- Communications: announcements, threaded discussions, direct messaging, push notifications

---

## 10. Mobile Guidelines

### 10.1 Stack

- React Native + Expo — Android and iOS from day one
- WatermelonDB for local offline storage and sync queue
- Sync queue pushes queued actions to school server on reconnect

### 10.2 Mobile-First Features

- Offline lesson access
- Offline assignment submission (queued, synced on reconnect)
- Offline attendance marking
- Push notification delivery via Firebase
- SMS fallback via local Zimbabwean gateways

### 10.3 Device Targets

- Low-end Android devices are the primary mobile target
- Optimise bundle size, image assets, and render performance accordingly

---

## 11. Testing Strategy

### 11.1 Required Test Coverage

- Unit tests for all domain services and business logic
- Integration tests for all API endpoints
- AI-generated modules must ship with tests — no exceptions

### 11.2 Load Testing Scenarios

- Enrolment spikes at term start
- Exam windows with concurrent quiz submissions
- Large file uploads (lesson content, assignments)
- Concurrent LAN usage by many devices simultaneously

### 11.3 Disaster Recovery

- Quarterly DR drills
- Backup restore tests to verify recovery point and time objectives

---

## 12. Three-Month Delivery Plan

| Weeks | Focus |
|---|---|
| **1–2** | Requirements, domain model, wireframes, ADRs, security baseline, master prompt construction |
| **3–4** | Local server infrastructure, Docker setup, CI/CD, auth, roles, multi-tenancy, school setup |
| **5–6** | Courses, enrolment, lessons, file upload, announcements |
| **7–8** | Assignments, quizzes, grading, messaging, notifications, reporting MVP |
| **9–10** | AI analytics MVP: engagement dashboard, risk scoring, recommendations |
| **11–12** | Load testing, security audit, accessibility fixes, DR drills, mobile polish, deployment |

---

## 13. MVP Scope

The following must ship at launch:

- Users and roles
- Student information management
- Courses, classes, and timetable management
- Attendance tracking
- Assignments, quizzes, and grading
- Gradebook and report cards
- Fee management and invoicing
- Notifications — in-app, SMS, and email
- Admin and teacher dashboards
- Basic AI risk and recommendation insights
- Web app (Next.js)
- Mobile app — Android and iOS (React Native + Expo, offline-first)

---

## 14. Deferred Features

Not in MVP. Introduced in later phases:

- Advanced proctoring
- Deep adaptive learning
- Complex marketplace and billing workflows
- Library management
- Full cloud deployment and enterprise monitoring (Kubernetes, ELK, Prometheus, Grafana)
- Microservices extraction (video processing, search, AI inference, notifications)

---

## 15. Multi-Agent Development Workflow

### 15.1 Tool Responsibilities

| Tool | Best Used For |
|---|---|
| **Claude / ChatGPT** | Architecture decisions, schemas, API contracts, threat models, planning docs |
| **Claude Code** | Generate and refactor whole modules; multi-file reasoning |
| **GitHub Copilot** | Inline code completion, small refactors, filling methods and boilerplate |
| **Cursor** | Fast interactive implementation inside the IDE |
| **ChatGPT (data analysis)** | Debugging plans, SQL, test design, data analysis |

### 15.2 Rules

1. Lock coding standards before any code generation begins.
2. Build a **master system prompt** encoding the full architecture, stack, domain model, and coding standards. All agents work from this single source of truth.
3. Generate code from ADRs and API contracts — not from vague descriptions.
4. Require tests with every AI-generated module.
5. Run lint, SAST, dependency scan, and human review on all AI-generated code.
6. Keep a small set of approved patterns so agents do not drift.

---

## 16. Post-Launch Operations

- Monthly security patching
- Quarterly security review
- AI model monitoring and retraining schedule
- Backup restore tests (verify recovery)
- Dependency updates and container rescans
- Ministry of Education reporting compliance checks

---

*Module decomposition diagrams and DDD context maps are maintained in separate documents.*
