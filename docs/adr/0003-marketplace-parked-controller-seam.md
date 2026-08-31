# ADR-0003: Marketplace parked; controller seam kept open

- Status: Accepted
- Date: 2026-06-27
- Deciders: platform architecture
- Provenance: conversation `5985639b-3007-40c8-a98e-2c07dce7b223`
- Related: PRD §8 (Non-Goals), §29 (Controller Runtime), §30 (Example Controller
  Patterns)

## Context

A platform requirement is an **apps marketplace** where tenant owners integrate
external services with the platform.

PRD §8 lists "provide a public third-party marketplace" as an explicit **v1
non-goal**. Building a marketplace now would pull vNext toward third-party
extensibility before the core control plane (tenants, IAM, RBAC, entitlements,
audit) is proven against the first product (HR OS Attendance, PRD §38).

Structurally, a marketplace integration is a textbook reconciliation problem: a
desired-state resource (an installation request) whose controller calls an
external system, stores credentials/handles, registers webhooks, and reports
`status.conditions` — exactly the cert-manager / external-dns pattern the PRD
already anticipates (§30).

## Decision

**Park the marketplace for v1.** Build no marketplace resources or controllers
now.

**Keep the seam open:** ensure the controller runtime (PRD §29) and the
resource model (metadata/spec/status, conditions, finalizers, events,
list/watch) can express a future marketplace **without architectural change**.
When it returns, it should be expressible as ordinary resources plus a
controller, for example:

- `AppListing` — a catalog entry (cluster-scoped).
- `AppInstallation` — an organization-scoped desired-state resource; a controller
  reconciles it: call the external API, persist a credential **reference** (never
  a secret in spec/labels/annotations, PRD §13.5), register webhooks, and report
  `status.conditions` + `observedGeneration`.
- Finalizers handle clean external teardown on delete (PRD §20.5).

These are illustrative shapes to validate the seam, **not** committed designs.

## Consequences

### Positive

- Keeps v1 focused on the control-plane core and the first product proof.
- No premature third-party API surface, security review burden, or partner
  contract surface.
- The reconciliation model that the marketplace will need is exercised earlier by
  other controllers (e.g., entitlement snapshot), so the seam is validated for
  free.

### Negative / risks

- The requirement remains unmet until a product needs it. Acceptable: no current
  v1 reference consumer requires it (the dental marketplace starts as a modular
  monolith per PRD §6.5).

### Follow-ups

- Revisit when a product (e.g., Dental Procurement OS, or any product needing
  external integrations) makes it a real requirement. That work would add new
  resources + a controller and supersede the "parked" status of this ADR.

## Alternatives considered

- **Design it now (resources + controller spec), build later.** Rejected:
  locking the marketplace shape before any product validates the controller
  runtime risks designing the wrong abstraction. Better to let real controllers
  prove the seam first.
- **Drop from roadmap entirely.** Rejected: the requirement is real for the
  platform's long-term vision; parking with a known integration pattern preserves
  it at near-zero cost.
