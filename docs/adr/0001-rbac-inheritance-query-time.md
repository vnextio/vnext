# ADR-0001: RBAC inheritance via query-time ancestor check

- Status: Accepted
- Date: 2026-06-27
- Deciders: platform architecture
- Provenance: conversation `5985639b-3007-40c8-a98e-2c07dce7b223`
- Related: PRD §10.1 (Core Resource Scope), §24 (RBAC), §31.4 (Cross-Tenant Safety)

## Context

vNext models a fixed, shallow tenant hierarchy:

```text
platform -> organization -> scope
```

This is enforced in code today: a `Tenant` carries `spec.kind` and
`spec.parentTenantName`, and `internal/core/tenant` validates that organizations
parent to a platform and scopes parent to an organization.

`RoleBinding` is scoped to an organization **or** a scope (PRD §10.1). The PRD
does **not** specify whether a binding granted at the organization level applies
to the scopes beneath it. Without a decision, "org admin can administer their
branches" is undefined behavior.

Two viable strategies:

1. **Query-time ancestor check** — at authorization time, a permission is
   granted if the principal has a satisfying `RoleBinding` at the requested scope
   **or** at any ancestor (its organization, or the platform).
2. **Propagation controller (HNC-style)** — a controller materializes
   organization-level bindings into per-scope bindings; authorization then only
   checks the exact scope.

## Decision

For v1, use **query-time ancestor checking**.

Authorization resolves an action on a resource in scope `S` by checking, in order
of the resolved scope chain `S -> org(S) -> platform`, whether the principal (or
service account) has a `RoleBinding` granting the required permission at any level
of that chain. The first satisfying binding grants access; absence denies
(default deny, per PRD §31.4).

Because the hierarchy is at most three levels deep, the ancestor chain is bounded
and cheap to resolve. Results may be cached per `(principal, scope)` with
invalidation on `RoleBinding`/`Membership` writes.

## Consequences

### Positive

- **Strong consistency.** A binding change takes effect on the next request; no
  reconciliation lag and no risk of drift between desired and materialized rows.
- **No new controller, no derived rows.** Less storage, fewer moving parts, and
  no propagation bugs (orphaned/duplicated child bindings).
- **Simple mental model** for the first product integration (HR OS Attendance).

### Negative / risks

- Every authorization decision walks the ancestor chain. Mitigated by the bounded
  depth and an effective-permission cache.
- If the hierarchy ever deepens beyond three levels, walk cost grows linearly;
  revisit then.

### Follow-ups

- The authorization service must expose the resolved scope chain so audit events
  can record **which** binding (and at which level) granted access (PRD §32.1).
- Effective-permission caching and its invalidation triggers are a separate
  implementation concern, not part of this ADR.

## Alternatives considered

- **Propagation controller (HNC-style).** Rejected for v1: it trades strong
  consistency for eventual consistency and adds a controller plus derived rows,
  none of which the shallow hierarchy justifies yet. Revisit only if a future
  product needs deep hierarchies or if flat-binding read performance becomes a
  bottleneck. Such a change would supersede this ADR.
- **No inheritance (exact-scope bindings only).** Rejected: organization
  administrators would need an explicit binding in every scope, which is poor UX
  and error-prone for the org/branch products (HR, Healthcare, Maritime).
