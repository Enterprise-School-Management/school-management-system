# Architectural Decision Records (ADRs)

This directory captures every significant architectural decision made for the School Management System. Each file is an ADR — a short document that records the context, the decision, and its consequences so that future contributors understand *why* the system is shaped the way it is, not just *what* it is.

## Why ADRs?

Decisions fade from memory. ADRs are cheap to write and invaluable six months later when someone asks "why do we use X instead of Y?"

## File naming

```
NNNN-short-kebab-title.md
```

- `NNNN` is a zero-padded sequential number starting at `0001`.
- `0000-template.md` is the blank template — copy it, never edit it.

## Status vocabulary

| Status | Meaning |
|--------|---------|
| `Draft` | Being written; not yet decided. |
| `Accepted` | Decision is in force. |
| `Deprecated` | Superseded but kept for history. Reference the superseding ADR. |
| `Superseded by ADR-NNNN` | Explicitly replaced; link to the new record. |

## Index

| ADR | Title | Status |
|-----|-------|--------|
| [0001](0001-superadmin-authentication.md) | Superadmin Authentication Strategy | Accepted |
| [0002](0002-redis-cache-and-broker.md) | Redis Dual-Use: Cache and Message Broker | Accepted |

## Process

1. Copy `0000-template.md` to a new numbered file.
2. Fill in all sections. Leave nothing blank — write "N/A" if a section truly does not apply.
3. Open a PR. ADRs are reviewed like code — they need at least one human approval.
4. Once merged, add a row to the index table above.
5. Reference the ADR from `02-ChangeLog.md` when it lands.
