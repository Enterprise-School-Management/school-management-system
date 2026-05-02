# ADR-0001 — Superadmin Authentication Strategy

**Date:** 2026-05-02  
**Status:** Accepted  
**Deciders:** system_designer, security_officer

---

## Context

The system supports a **superadmin** role that monitors all schools from a single installation — the Superadmin Dashboard. The key question is how this role authenticates:

1. A **separate login page / Django site** exclusively for platform operators.
2. A **flag on the existing auth system** — the same `POST /auth/login` endpoint, with an extra claim in the JWT.
3. A **separate identity provider** (Keycloak, Auth0) for the platform operator while schools use Django Auth.

The deployment target is a resource-constrained on-premises school server or a small cloud VM. Maintaining two separate auth systems or a full Keycloak instance is operationally heavy and counter to the "Zimbabwe-context fit" principle.

The system already uses Django Auth + JWT for all other roles. The 13 roles (defined in `SchoolMembership.role`) are managed through a single `SchoolMembership` model.

## Decision

Superadmin access is **a role on the same auth system** — the same `POST /api/v1/auth/login` endpoint used by all other users.

Specifically:

- A `User` record is created for the platform operator as normal.
- A `SchoolMembership` row is created with `role = system_designer` (or `security_officer`) and `is_superadmin = True`. The `school_id` on this membership references a reserved **platform tenant** (`short_code = __platform__`) that is created on first install and never represents a real school.
- On login, the JWT issued for a superadmin user includes an additional claim: `"superadmin": true`.
- Django middleware checks for this claim before routing to any `/api/v1/super/*` endpoint. Requests without `superadmin: true` in the token receive `403 Forbidden`.
- The superadmin login URL is the same as the regular login URL. There is no separate Django site. Security is achieved via the `is_superadmin` flag and the restricted `/super/*` route prefix.

## Alternatives considered

| Alternative | Why rejected |
|-------------|--------------|
| Separate Django site (`SITE_ID = 2`) | Doubles operational complexity (separate process, separate DB migrations for auth tables, separate cookie domain). Does not fit the "single Docker Compose" deployment model. |
| Keycloak / Auth0 for platform operator | Adds a heavyweight external dependency that cannot run offline or on low-spec hardware. Deferred to the cloud-hosted multi-school platform phase. |
| Separate login page with shared DB | UX is different but the backend would be the same JWT system. Provides no additional security over a claim-based check while adding UI surface. |

## Consequences

### Positive

- Single auth code path — no duplication of login, token refresh, or password-reset flows.
- Superadmin credentials are managed with the same tooling (Django admin or the `/users` API) as regular staff credentials.
- Deployable in a single Docker Compose stack with no additional services.

### Negative / trade-offs

- The `is_superadmin` flag must be protected from accidental self-promotion. Only the Django admin (`manage.py shell` or a migration fixture) can set it; the REST API never exposes this field as writable.
- The `/super/*` route prefix must be enforced in **both** the DRF permission class and the tenant middleware — two places that must stay in sync.

### Risks and mitigations

| Risk | Mitigation |
|------|-----------|
| A compromised regular user account escalates to superadmin | `is_superadmin` is never writable via the public API. Setting it requires Django admin access or a direct DB change. |
| Token replay gives superadmin access after the flag is revoked | Token TTL is short (15 min access tokens). Revoking the `SchoolMembership.is_superadmin` flag invalidates future tokens at the next refresh. |
| `/super/*` routes accidentally reachable without the claim check | A dedicated DRF permission class (`IsSuperadmin`) is applied at the router level, not per-view — missing it on a new view is a code-review failure. |

## References

- [01-Architecture.md §7 — Multi-Tenancy](../01-Architecture.md)
- [03-ApiReference.md §11 — Superadmin endpoints](../03-ApiReference.md)
- [08-SecurityGuide.md — Permission model](../08-SecurityGuide.md)
- [06-DomainModel.md — SchoolMembership entity](../06-DomainModel.md)
