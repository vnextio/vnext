# ADR-0007: Kubernetes substrate, control/operational plane split, and `vnext-service`-owned migrations

- Status: Accepted
- Date: 2026-06-28
- Deciders: platform architecture
- Provenance: conversation `8256c41a-13f8-462d-af8e-8ee52d9dd369`
- Amends: PRD §14 (Storage), §15 (Control-Plane Data Model), §21.2 (Leases), §35 (Migration from Current Kernel)
- Related: ADR-0005 (build vNext), ADR-0006 (aggregation)

## Context

vNext will be **deployed strictly on Kubernetes** (k3d / k3s / k8s, any variant).
That fact forces decisions the PRD made under a "no etcd, no required orchestrator"
assumption (§21.2 built a Postgres lease table specifically so "etcd is not
required"). It also introduces a terminology need (one word for the backends that
extend the platform) and a decision about who runs domain migrations.

A guardrail must be stated loudly: **vNext resources are not Kubernetes CRDs.**
`Tenant`, `AttendanceRecord`, etc. live in Postgres and are served by the vNext
API server. Kubernetes is the *infrastructure substrate* that runs vNext; it does
**not** model SaaS types (PRD §8.3 still holds). "Deploy on k8s" must never drift
into "model domain types as CRDs."

## Decision

### Terminology: `vnext-service`

`vnext-service` is the umbrella term for any backend that registers with vNext to
serve types (via `APIService`) and/or run controllers. It supersedes the muddled
"module / domain API server / controller" trio. A `vnext-service` may serve types,
reconcile them, or both.

### Two planes

| Plane | Contents | Lives in | Change vs PRD |
| --- | --- | --- | --- |
| **Control-plane data** | tenants, principals, resources, audit, `resource_events` | **Postgres** | none |
| **Operational plane** | config, secrets, service discovery, leader election | **Kubernetes-native** | new |

- **Configuration & secrets:** a base **ConfigMap** (DB host/name, cache/NATS
  endpoints, service registry) + **Secret** (DB creds, JWT signing keys, provider
  client secrets) are consumed by every `vnext-service`, with per-service
  env/overlay overrides. The shared config supplies the *connection*; each
  `vnext-service` still owns its *schema* (PRD §14.3).
- **Service discovery:** `vnext-service` endpoints resolve via Kubernetes
  **Service + DNS**, recorded in vNext's `APIService` registry (ADR-0006).
- **Leader election:** use Kubernetes-native `coordination.k8s.io/Lease`. The
  Postgres `control_plane.leases` table (PRD §21.2) is **retired**.

### Migrations are `vnext-service`-owned

- The **vNext control plane migrates only `control_plane.*`** (its own
  primitives). It ships and runs those migrations itself.
- **Each `vnext-service` owns and runs migrations for its own schema(s)**
  (`attendance.*`, `appointments.*`). A service self-migrates on deploy
  (init container / Kubernetes `Job`) before it serves.
- The control plane does **not** orchestrate, sequence, or gate domain migrations.
  This is the persistence-lifecycle expression of "domain = k8s-style extension"
  (PRD §5.7).
- **DB-level least privilege:** each `vnext-service` receives a **schema-scoped
  role** able to DDL only its own schema, so schema ownership is enforced at the
  Postgres grant level, not by convention.
- **No cross-schema hard foreign keys** (PRD §14.4 hardens from guideline to
  rule): with independent, unordered migrations, no service's migration may assume
  another service's table exists. Cross-schema references stay logical (UUID),
  validated via the API/admission.

This **amends PRD §35**, which listed central "migration orchestration" as a
kernel pattern to preserve. Central orchestration is *not* preserved. What is
preserved is migration *tooling/ergonomics*, packaged into the `vnext-service`
SDK so each service gets first-class migration DX **without** the control plane
running it.

## Consequences

### Positive

- Less bespoke machinery: leader election, config, secrets, and service discovery
  use battle-tested k8s primitives instead of hand-rolled equivalents.
- Independent, ungated service deploys (including migrations) keep "extend without
  rebuilding the core" true.
- Schema ownership becomes a real boundary (DB grants), not a guideline.

### Negative / risks

- vNext is now coupled to a Kubernetes substrate for operations (portability to
  bare VMs/ECS is given up). Accepted: deployment is strictly k8s by decision.
- Control-plane *data* remains in Postgres, so the system spans two stores (k8s
  for ops, Postgres for data). Acceptable and intentional — they are different
  layers.

### Follow-ups

- Define the base ConfigMap/Secret keys and the per-service override convention.
- Define the schema-scoped DB role/grant template provisioned per `vnext-service`.
- Settle the deployment-naming convention (the deferred `${PROJECT}` prefix — an
  ops/namespace label only, **not** a data-model concept, since `Project` is
  killed per PRD §9).

## Alternatives considered

- **Keep Postgres leases + a bespoke registry for portability (PRD §21.2).**
  Rejected given the strict-k8s decision: it rebuilds primitives the substrate
  already provides.
- **Central migration orchestration (kernel pattern, PRD §35).** Rejected:
  reintroduces a deploy-time coupling between the control plane and every domain
  service, contradicting the extension model.
- **Model domain types as Kubernetes CRDs.** Rejected: violates PRD §8.3; k8s is
  the substrate that runs vNext, not the data model for SaaS types.
