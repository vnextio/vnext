# ADR-0005: Build vNext fresh; harvest the kernel; strangler migration

- Status: Accepted
- Date: 2026-06-28
- Deciders: platform architecture
- Provenance: conversations `5985639b-3007-40c8-a98e-2c07dce7b223`, `8256c41a-13f8-462d-af8e-8ee52d9dd369`
- Related: PRD §3 (Background), §5 (Core Principles), §35 (Migration from Current Kernel), RFD-0001 (Superseded)

## Context

An existing framework, `edgescaleDev/kernel` (+ `sdk`), is a mature **modular
monolith**: topological module boot, schema-per-module + RLS, a pluggable
identity-provider chain (issuer routing already implemented), a transactional
outbox, manifest-declared RBAC, per-tenant module activation, audit, telemetry.
RFD-0001 originally proposed *evolving* this kernel into the control plane.

The original intent behind the kernel was twofold:

1. **One reusable framework** so each new SaaS startup is not built from scratch
   (Laravel-style reuse).
2. **Kubernetes-style extensibility** so the platform can be extended with
   services/controllers like k8s is extended with CRDs/controllers.

The kernel delivers (1) but structurally cannot deliver (2). Its defining
constraint — *a new module means recompiling and redeploying the binary* — is
load-bearing, not incidental. The architecture chosen across ADR-0004/0006/0007
(aggregating resource-oriented API server, unified discovery, baked-in auth,
dynamic types, k8s substrate) is a different paradigm from the kernel's
imperative, handler-oriented, single-binary model. Retrofitting it onto the
kernel yields a monolith cosplaying as a control plane, with two mental models
colliding.

## Decision

**Build vNext as a new repository (`github.com/vnextio/vnext`). Treat
`kernel`/`sdk` as a parts donor, not a foundation to evolve.**

### Harvest from the kernel (port the proven pieces)

- The **identity-provider chain / issuer routing** → feeds the ADR-0004
  authentication chain.
- The **transactional outbox** → the `resource_events` / change-feed backbone.
- **IAM tenant tree + RBAC modes + per-tenant auth config** → re-expressed as
  `Tenant` / `RoleBinding` / `AuthProvider` resources (port the *logic*).
- **Migration tooling/conventions, schema-per-owner + RLS, telemetry / health /
  bootstrap, CLI patterns** → lift as packaged libraries/SDK.

### Deliberately leave behind (per PRD §35)

- Modules compiled into one product binary.
- The in-process module registry as *the* architecture.
- In-process cross-module readers.
- The generic `any`-client as the primary SDK.

### De-risk with a strangler migration

1. Build the vNext core slice and prove it end-to-end with exactly one path —
   HR OS Attendance (PRD §33.5 / §38).
2. Keep the kernel serving whatever it serves today; vNext does not replace
   existing backends immediately (PRD §8.9).
3. Migrate strategic products onto vNext as it earns trust; let non-strategic
   products ride the kernel until/unless they need what vNext provides.

## Consequences

### Positive

- Delivers both original intentions by placing each on its correct layer
  (platform = reused like a framework; domain = extended like k8s) — see PRD §5.7.
- Contained downside: a stumbling vNext does not take the business down, because
  existing products keep running on the kernel during migration.
- The new architecture is built clean, not fought into an incompatible base.

### Negative / risks

- A new control plane is a substantial build (the MVP resource set + apiserver +
  authn/authz + admission + discovery + watch + client + controller runtime).
  Accepted: the kernel's "faster" path is a local optimum that hits the exact
  ceiling the strategic roadmap must cross.
- Second-system risk (over-building, re-deriving fixed bugs). Mitigated by
  aggressive harvesting and a single-product proof before scaling.

### Follow-ups

- Maintain an explicit harvest checklist mapping kernel packages → vNext targets.
- Define the deprecation/criteria gate for migrating each product off the kernel.

## Alternatives considered

- **Evolve the kernel in place (RFD-0001).** Rejected: the rebuild-on-new-module
  ceiling is structural; removing it means rebuilding the core anyway, while every
  other kernel assumption resists the resource-oriented model.
- **From-zero rewrite ignoring the kernel.** Rejected: discards proven primitives
  (identity chain, outbox, migrations, RLS) and maximizes second-system risk.
- **Kong/API-gateway + independent services instead of a control plane.**
  Rejected: a gateway unifies *traffic*, not *semantics*; it cannot provide
  discovery-by-kind, a uniform resource/RBAC model, or a uniform client, so the
  shared platform would be re-implemented inconsistently across services. A
  gateway/ingress remains complementary at the edge (ADR-0007), not a substitute.
