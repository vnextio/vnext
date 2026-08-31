# ADR-0002: Per-tenant AuthProvider resource; BFF verifies tokens

- Status: **Superseded by ADR-0004**
- Date: 2026-06-27
- Deciders: platform architecture
- Provenance: conversation `5985639b-3007-40c8-a98e-2c07dce7b223`
- Related: PRD §23 (Identity, Auth, and Delegation), §23.1 (External Auth),
  §23.4 (Request Context)

> **Superseded by [ADR-0004](0004-authentication-baked-in.md) (2026-06-28).**
> This ADR kept token verification in the BFF and treated `AuthProvider` as
> BFF-read config. The project subsequently adopted the kube-apiserver model
> (ADR-0006), where authentication is a property of the API server. ADR-0004
> bakes authentication into vNext: `iss`-keyed central + per-tenant
> `AuthProvider`, `iss`+`aud` validation, and TokenReview/SAR + front-proxy
> delegation. The **`AuthProvider` resource itself survives** (carried into
> ADR-0004); only the "BFF verifies, vNext holds no keys" stance is reversed.

## Context

A platform requirement is: a **central** auth provider plus **per-tenant /
per-scope** auth providers, where business owners can integrate multiple
identity providers (Firebase, Keycloak, Zitadel, Auth0, custom OIDC, etc.).

PRD §23.1 deliberately decides that **vNext is not the external identity
provider** — product BFFs verify external tokens and normalize them into a vNext
principal/membership context. That keeps vNext out of per-IdP crypto and token
formats.

But §23.1 as written leaves the requirement above without a home: there is no
place that records *which* providers a given organization is allowed to use, or
the per-provider configuration a BFF needs to verify a token (issuer, JWKS URL,
audience, allowed sign-in methods).

The current kernel already solved the *policy* half of this with
`TenantAuthConfig` (a per-tenant provider allowlist with a `config` JSONB blob:
open by default, allowlist once the first row exists). That pattern is worth
carrying forward, expressed as a first-class resource.

## Decision

Introduce an **`AuthProvider` resource** owned by vNext, while **token
verification stays in the product BFF** (preserving PRD §23.1).

- **Group/kind:** `iam/v1`, `kind: AuthProvider`.
- **Scope:** cluster-scoped instances define platform defaults / available
  providers; organization-scoped instances define per-tenant policy and config.
  Resolution is most-specific-wins (org overrides platform default).
- **Spec (illustrative, not final):**
  - `type` — `oidc`, `firebase`, `apikey`, etc.
  - `issuer`, `jwksURL`, `audience`
  - `allowedSignInMethods`
  - `enabled`
  - non-secret config; **secrets are never stored in spec/labels/annotations**
    (PRD §13.5), they are referenced.
- **Semantics:** "open by default, allowlist once configured" — if an
  organization has no `AuthProvider`, platform defaults apply; once it has one or
  more, only those providers are accepted for that organization.
- **Consumer flow:** a BFF reads the applicable `AuthProvider`(s) via the
  generated client to learn how/whether to accept a given token, performs the
  verification itself, then resolves the vNext principal/membership context
  (PRD §23.4) and proceeds with signed delegation downstream (PRD §23.3).

vNext does **not** perform JWT signature verification or hold IdP signing keys.

## Consequences

### Positive

- Satisfies the "central + per-tenant, multiple providers" requirement
  declaratively, with the same admission/audit/list/watch machinery as every
  other resource.
- Keeps vNext free of per-IdP crypto and token-format coupling (honors §23.1).
- Carries forward the proven `TenantAuthConfig` allowlist semantics.

### Negative / risks

- Verification logic lives in each BFF. Mitigated by shipping a shared
  verification helper/middleware that consumes `AuthProvider` so BFFs do not each
  reimplement it.
- Secret material (client secrets, signing keys) must live in a secret store and
  be referenced from `AuthProvider`, never inlined. Admission must reject
  detectable secrets (PRD §13.8).

### Follow-ups

- This resource is **not** in the original MVP resource list (PRD §33.1).
  Sequencing decision: it can be added in the IAM slice or deferred until a
  second IdP is actually onboarded. Track as an MVP-scope amendment.
- Define the secret-reference mechanism (out of scope for this ADR).

## Alternatives considered

- **BFF-only, strictly per PRD §23.1 (no vNext state).** Rejected: leaves
  per-tenant provider policy/config with no platform home, so the multi-provider
  requirement cannot be centrally administered, audited, or listed.
- **vNext verifies tokens (identity chain in the control plane).** Rejected:
  contradicts §23.1 and couples vNext to every IdP's token format and key
  rotation. The current kernel's `IdentityProviderChain` lives at the edge here,
  not in the control plane.
