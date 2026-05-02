# 02 — Change Log

Versioned change history for the School Management System. Follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) conventions. Dates use `YYYY-MM-DD`.

**See also:** [11-Roadmap.md](11-Roadmap.md) · [13-ProjectStatus.md](13-ProjectStatus.md)

---

## Template for New Entries

Copy this block to the top of the file (under "Unreleased") when logging a change.

```markdown
## [Unreleased]

### Added
- …

### Changed
- …

### Deprecated
- …

### Removed
- …

### Fixed
- …

### Security
- …

### Operations
- … (deployments, DR drills, incident notes)
```

Rules:
- One bullet per discrete change.
- Reference `BR-*` codes for business-rule changes.
- Reference ADR numbers when an architectural decision lands.
- Security incidents go under **Security** with date and a one-line impact statement.

---

## [Unreleased]

### Added
- Documentation scaffolding in `.claude/docs/` — sixteen files covering architecture, domain, API, database, security, testing, development, deployment, training, business rules, roadmap, project status, changelog, quick reference, glossary, and index.

---

## [0.0.1] — 2026-04-09

### Added
- Initial Django 6.0.4 scaffold (`manage.py`, `school_management_system/` config, SQLite default DB).
- Canonical technical specification at [docs/spec.md](../../docs/spec.md) v2.0.
- Thirteen role agent definitions under [.claude/agents/](../agents/).

---

## Release Tags

Tags will follow `vMAJOR.MINOR.PATCH`. Pre-1.0 releases track the roadmap phases:

| Tag | Meaning |
|-----|---------|
| `v0.1.0` | End of Phase 0 — foundation complete |
| `v0.2.0` | End of Phase 1a — identity & tenancy |
| `v0.3.0` | End of Phase 1b — academic core |
| `v0.4.0` | End of Phase 1c — assessments & finance (MVP) |
| `v0.5.0` | End of Phase 2 — AI MVP |
| `v1.0.0` | End of Phase 3 — hardening & production launch |

Detail: [11-Roadmap.md](11-Roadmap.md).
