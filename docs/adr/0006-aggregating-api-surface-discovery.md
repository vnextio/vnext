# ADR-0006: Aggregating API surface with unified discovery

- Status: Accepted
- Date: 2026-06-28
- Deciders: platform architecture
- Provenance: conversation `8256c41a-13f8-462d-af8e-8ee52d9dd369`
- Supersedes: PRD §37.1 #5 (v1 does not aggregate; BFFs call domain APIs directly)
- Amends: PRD §37.1 #4 (deferral of dynamic types / ResourceDefinition)
- Related: PRD §17–§19 (API-server semantics, discovery, list/watch), §22 (Resource Definitions and API Services), §28 (Client Model), ADR-0004, ADR-0007

## Context

PRD §37.1 #5 resolved that v1 would **not** aggregate: BFFs would call domain API
servers directly using generated clients, and domain servers would merely
*register* in discovery. That requires every consumer to know which server owns
which type.

The chosen contract is the opposite, and it is the kube-apiserver contract:
**consumers must not know which backend serves a type.** They discover types the
way `kubectl api-resources` does — from one endpoint — and interact uniformly,
whether a type is built-in, a registered dynamic type, or served by an external
`vnext-service`. This makes direct-to-domain calls untenable (they leak backend
topology to clients) and promotes dynamic types from "deferred" to a day-one
architectural invariant.

## Decision

**vNext is the single API surface and the single discovery surface.** It
aggregates; consumers are backend-agnostic.

### One discovery tree

vNext serves `GET /apis`, `GET /apis/{group}/{version}`, and an `api-resources`
equivalent. Discovery advertises groups, versions, kinds, verbs, scope,
status-subresource support, and watch support — for *all* types regardless of
backend.

### Three serving mechanisms, indistinguishable to the client

| Mechanism | k8s analog | Serves | Backed by | Example |
| --- | --- | --- | --- | --- |
| **Built-in typed** | core/apps types | first-party core types | vNext's own typed tables | `Tenant`, `Principal`, `AuthProvider`, `RoleBinding` |
| **Registered dynamic (CRD-style)** | CRD / apiextensions | low/medium-volume desired-state/config | vNext generic store + `ResourceDefinition` | `Certificate` (cert-manager-style), integration config |
| **Aggregated** | `APIService` → extension server | high-volume / query-heavy domain types | external `vnext-service`s, **proxied** through vNext | `attendance-records`, `appointments` |

The client uses a **typed client** for types it was generated against and a
**dynamic client** for the rest. Discovery is what makes backend-agnostic access
possible.

This preserves the PRD's anti-bloat rule (§5.4 / §8.17): high-volume domain data
does **not** go in the generic store; it lives in `vnext-service`s and reaches the
client via aggregation/proxy, not via CRD-style generic storage.

### Read path — strict proxy with governed caching

Reads for a type are routed to its owning backend (resolved via discovery), not
served from a side channel. Two caches with different safety profiles:

- **Discovery cache (GVK → owning `vnext-service`).** Always on; invalidated only
  when an `APIService`/`ResourceDefinition` registration changes. (k8s caches
  discovery the same way.)
- **Result cache (the actual rows).** **Per-type opt-in only**, with a declared
  TTL, plus invalidation driven by the `resource_events` stream
  (resourceVersion-keyed). Blanket-caching mutable domain lists at the proxy is
  forbidden — it serves stale reads. Historical/immutable slices may cache
  freely; live data uses a short TTL or none.

Pages are bounded (e.g. 100–200 rows) and vNext scales horizontally (stateless),
so the strict-proxy read path does not make vNext an OLTP chokepoint.

### Authentication across the boundary

Aggregation requires the API server to authenticate (ADR-0004) and forward
trusted identity to the owning `vnext-service` (front-proxy headers), or let the
service delegate via TokenReview/SubjectAccessReview. This is what lets
`vnext-service`s stay ignorant of identity.

### MVP phasing

The *architecture* (discovery contract, client model, aggregation seam) is
designed from day one. *Implementation* of the generic store (dynamic types) and
the proxy can be phased after the core slice — but the discovery contract and
client are built assuming them, so the design is not walled in.

## Consequences

### Positive

- Consumers depend on one endpoint and one client; backend topology is free to
  change without breaking clients.
- The single write surface enforces admission/RBAC/audit/resourceVersion
  uniformly for every type.
- Dynamic/CRD-style extensibility and aggregated domain services share one
  discovery + client model.

### Negative / risks

- vNext is on the hot read path for aggregated types; mitigated by bounded pages,
  horizontal scaling, governed result caching, and discovery caching.
- Aggregation/proxy + front-proxy trust is real machinery to build (more than the
  superseded direct-call model). Accepted: it is the cost of the backend-agnostic
  invariant the platform requires.

### Follow-ups

- Specify the discovery payload schema and the `api-resources` equivalent.
- Specify the aggregation/proxy transport, the front-proxy header set, and the
  per-type result-cache declaration (cacheable + TTL) in registration.
- Sequence generic-store (dynamic types) implementation after the core slice.

## Alternatives considered

- **Direct-to-domain calls (PRD §37.1 #5).** Superseded: leaks backend topology to
  clients, breaking the "consumers don't know which service owns a type"
  invariant.
- **Proxy writes, escape-hatch reads (RFD Write/Read rule).** Considered and
  rejected as the default: a single read surface with governed caching keeps the
  invariant intact; per-type opt-in caching addresses the performance concern
  without a separate read channel. (A read projection remains available to a
  `vnext-service` internally, but is not a client-facing bypass.)
- **Everything as CRD-style generic storage.** Rejected: violates the anti-bloat
  rule for high-volume domain data (PRD §5.4 / §8.17).
