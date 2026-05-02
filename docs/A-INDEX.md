# Documentation Index

Master navigation and reading guide for the School Management System documentation set. Every doc in `.claude/docs/` is listed here; add a row whenever a new file is introduced.

---

## Documentation File Directory

| File | Purpose |
|------|---------|
| **A-INDEX.md** | *(this file)* Master navigation and reading guide |
| [B-QUICK_REFERENCE.md](B-QUICK_REFERENCE.md) | At-a-glance cheat sheet: commands, URLs, roles, conventions |
| [C-Glossary.md](C-Glossary.md) | Domain, workflow, finance, and technical term definitions |
| [01-Architecture.md](01-Architecture.md) | Layer diagram, module responsibilities, dependency rules, tech choices |
| [02-ChangeLog.md](02-ChangeLog.md) | Versioned change history with template for new entries |
| [03-ApiReference.md](03-ApiReference.md) | REST endpoint reference for all domain workflows |
| [04-DevelopmentGuide.md](04-DevelopmentGuide.md) | Local setup, conventions, migrations, contribution workflow |
| [05-TrainingManual.md](05-TrainingManual.md) | Step-by-step workflows for the 13 system roles |
| [06-DomainModel.md](06-DomainModel.md) | Entities, properties, enums, relationships, state machines |
| [07-DatabaseGuide.md](07-DatabaseGuide.md) | Schema design, table names, FK rules, naming conventions, migrations |
| [08-SecurityGuide.md](08-SecurityGuide.md) | Auth, RBAC permission matrix, audit trail, secrets management |
| [09-TestingGuide.md](09-TestingGuide.md) | Testing strategy, priority test areas, setup, patterns |
| [10-DeploymentGuide.md](10-DeploymentGuide.md) | Docker packaging, environment config, migrations, rollback, hosting |
| [11-Roadmap.md](11-Roadmap.md) | Phased feature plan from foundation to production readiness |
| [12-BusinessRules.md](12-BusinessRules.md) | Operational, academic, and financial rules for the school management system |
| [13-ProjectStatus.md](13-ProjectStatus.md) | Current implementation status, locked decisions, immediate next actions |

---

## Reading Order

### New to the project — read in this order
1. [B-QUICK_REFERENCE.md](B-QUICK_REFERENCE.md) — orient yourself
2. [01-Architecture.md](01-Architecture.md) — understand the shape
3. [06-DomainModel.md](06-DomainModel.md) — learn the entities
4. [13-ProjectStatus.md](13-ProjectStatus.md) — see what exists today
5. [04-DevelopmentGuide.md](04-DevelopmentGuide.md) — set up your environment

### Building a feature
1. [06-DomainModel.md](06-DomainModel.md) — which entities are involved
2. [12-BusinessRules.md](12-BusinessRules.md) — constraints that must hold
3. [03-ApiReference.md](03-ApiReference.md) — existing or proposed endpoints
4. [08-SecurityGuide.md](08-SecurityGuide.md) — permission model
5. [09-TestingGuide.md](09-TestingGuide.md) — coverage expectations

### Operating the system
1. [10-DeploymentGuide.md](10-DeploymentGuide.md) — deploy and rollback
2. [08-SecurityGuide.md](08-SecurityGuide.md) — audit, access, incidents
3. [05-TrainingManual.md](05-TrainingManual.md) — support end-users

### Leading the project
1. [11-Roadmap.md](11-Roadmap.md) — phases and exit criteria
2. [13-ProjectStatus.md](13-ProjectStatus.md) — where we are now
3. [02-ChangeLog.md](02-ChangeLog.md) — what shipped when

---

## Source of Truth

The canonical technical specification lives at [docs/spec.md](../../docs/spec.md). Every file here derives from that spec plus the 13 role definitions under [.claude/agents/](../agents/). When this documentation and `docs/spec.md` disagree, `docs/spec.md` wins — update the doc in `.claude/docs/` to match and log the change in [02-ChangeLog.md](02-ChangeLog.md).

---

## Conventions

- Files prefixed `A`/`B`/`C` are navigation aids. Files prefixed `01`–`13` are the numbered reference set.
- Every file opens with a one-line purpose statement and a **See also** block.
- `TBD` means "design not yet settled." `Not yet implemented` means "design settled, code not written yet."
- Dates use `YYYY-MM-DD`.
- Emojis are not used.
