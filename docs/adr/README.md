# Architecture Decision Records

This directory records architecture decisions for vNext that go beyond, or
resolve open items in, [`../prd/0000-architecture.md`](../prd/0000-architecture.md).

The PRD is the broad architecture narrative. ADRs capture individual decisions —
their context, the choice made, the alternatives rejected, and the consequences —
so future contributors understand *why*, not just *what*.

## Governance & precedence

The governance docs live in three tiers:

- **`docs/adr/`** — individual, dated, immutable-once-accepted decisions.
- **`docs/prd/`** — the broad architecture narrative (the requirements/design).
- **`docs/rfd/`** — historical requests-for-discussion / design references.

**Precedence when documents conflict: ADR > PRD > RFD.** An ADR is the most
specific and most recent decision and wins. The PRD narrative is authoritative
where no ADR overrides it. RFDs are historical context only and never override
the PRD or an ADR.

## Conventions

- One decision per file, numbered `NNNN-kebab-title.md`.
- Status is one of: `Proposed`, `Accepted`, `Superseded by NNNN`, `Deprecated`.
- An ADR is immutable once `Accepted`. To change a decision, write a new ADR
  that supersedes it (and mark the old one `Superseded by NNNN`).

## Index

| ADR                                                          | Title                                                                              | Status                  |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------------- | ----------------------- |
| [0001](0001-rbac-inheritance-query-time.md)                  | RBAC inheritance via query-time ancestor check                                     | Accepted                |
| [0002](0002-authprovider-resource-bff-verifies.md)           | Per-tenant AuthProvider resource; BFF verifies tokens                              | **Superseded by 0004**  |
| [0003](0003-marketplace-parked-controller-seam.md)           | Marketplace parked; controller seam kept open                                      | Accepted                |
| [0004](0004-authentication-baked-in.md)                      | Authentication baked into the API server                                           | Accepted                |
| [0005](0005-build-vnext-fresh-kernel-as-donor.md)            | Build vNext fresh; harvest the kernel; strangler migration                         | Accepted                |
| [0006](0006-aggregating-api-surface-discovery.md)            | Aggregating API surface with unified discovery                                     | Accepted                |
| [0007](0007-kubernetes-substrate-plane-split-migrations.md)  | Kubernetes substrate, control/operational plane split, service-owned migrations    | Accepted                |

## Provenance

ADRs 0001–0003 were produced from the initial architecture design session.
ADRs 0004–0007 were produced from the continuation session, which also amended
the PRD to v0.4 and superseded RFD-0001.

Reference conversation IDs:
- `5985639b-3007-40c8-a98e-2c07dce7b223` (initial design session)
- `8256c41a-13f8-462d-af8e-8ee52d9dd369` (continuation session)
