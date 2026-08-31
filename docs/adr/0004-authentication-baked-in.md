# ADR-0004: Authentication baked into the API server

- Status: Accepted
- Date: 2026-06-28
- Deciders: platform architecture
- Provenance: conversations `5985639b-3007-40c8-a98e-2c07dce7b223`, `8256c41a-13f8-462d-af8e-8ee52d9dd369`
- Supersedes: ADR-0002 (Per-tenant AuthProvider resource; BFF verifies tokens)
- Related: PRD §23 (Identity, Auth, and Delegation), §24 (RBAC), ADR-0006 (Aggregating API surface)

## Context

ADR-0002 decided that **vNext does not verify tokens** — product BFFs verify
external tokens and `AuthProvider` is merely config the BFF reads. That decision
was made before the API-surface model was settled.

The project has since chosen the **kube-apiserver model**: vNext is *the* API
surface, and every consumer (BFF, controller, CLI, aggregated `vnext-service`)
talks to it through one client; consumers are deliberately ignorant of which
backend serves a type (ADR-0006). In that model, authentication cannot live in
each BFF — it must be a property of the API server itself, exactly as
kube-apiserver authenticates every request before authorization and admission.

The platform requirement is unchanged: a **central** auth provider plus
**per-tenant** auth providers, where business owners integrate multiple identity
providers (Firebase, Keycloak, Zitadel, Okta, Auth0, custom OIDC). Token
lifetimes vary per provider (e.g. Firebase ID tokens ~1h with longer-lived
refresh tokens; a tenant's Keycloak may use much shorter TTLs), which makes any
scheme that re-mints vNext-side assertions on top of provider tokens
(a second, mismatched TTL to manage) the wrong design.

## Decision

**Authentication is baked into the vNext API server.** vNext verifies the token
presented on every request, the way kube-apiserver does.

### Authentication chain (first-to-accept wins)

1. **ServiceAccount tokens** — vNext-issued, vNext-signed JWTs for
   machine-to-machine callers (BFFs, controllers, workers, `vnext-service`s).
2. **Central / platform OIDC** — one configured external provider, analogous to
   kube-apiserver's `--oidc-issuer-url`.
3. **Per-tenant OIDC** — each a first-class `AuthProvider` resource. This is the
   deliberate extension beyond vanilla Kubernetes, which allows only one
   cluster-wide OIDC.

### Provider selection — by issuer (`iss`)

The incoming token's `iss` claim selects the `AuthProvider` (central or
per-tenant) whose JWKS verifies the signature. This is a hash lookup, not
trial-verification across providers.

- `spec.issuer` is a **unique key** on `AuthProvider`; admission rejects a second
  provider claiming an already-registered issuer. Without this, `iss → provider`
  is ambiguous and becomes a tenant-confusion vulnerability.
- Verification asserts, in order: signature (via the provider's cached JWKS),
  then **`iss`**, then **`aud`**, then **`exp`**. `iss` selects *which keys*
  verify; `aud` proves the token was minted *for this deployment* — both are
  mandatory. (Validating `iss` without `aud` would accept any token from the same
  issuer minted for a different application.)

Verification is local once JWKS is cached — no network call on the hot path.

### Delegation across the aggregation boundary

The bespoke "BFF signs a delegation envelope" model is **dropped**. Two
k8s-native mechanisms replace it:

- **Front-proxy identity injection (primary).** Because vNext *is* the proxy
  (ADR-0006), it authenticates once and forwards to the owning `vnext-service`
  with trusted identity headers (the `--requestheader-*` / `X-Remote-User`
  model) over a mutually-authenticated channel. The `vnext-service` trusts those
  headers **only** from vNext. No per-call re-verification.
- **TokenReview / SubjectAccessReview (callback).** A `vnext-service` reached
  directly, or one wanting to independently verify, calls back to vNext's
  `TokenReview`/`SubjectAccessReview` endpoints. Same authority, opposite
  direction.

### Ownership boundaries

- **Token refresh is the client/BFF's responsibility, never vNext's.** vNext only
  *verifies* the token presented on a given request; it holds no refresh tokens
  and no sessions. When a provider token expires, the client refreshes and
  presents the new token. This keeps vNext out of OIDC session management.
- **Authentication results are cacheable** with a short TTL bounded by the
  token's `exp`, so high-QPS paths do not re-verify on every hop.

### `AuthProvider` resource

- **Group/kind:** `iam/v1`, `kind: AuthProvider`.
- **Scope:** cluster-scoped instances define platform-default / central
  providers; organization-scoped instances define per-tenant providers.
  Selection is by `iss`, not by scope precedence.
- **Spec (illustrative, not final):** `type` (`oidc`, `firebase`, …), `issuer`
  (unique), `jwksURL`, `audience`, `allowedSignInMethods`, `enabled`. Secrets are
  **referenced**, never inlined (PRD §13.5).
- **Semantics:** "open by default, allowlist once configured" carries forward
  from the kernel's `TenantAuthConfig`: an organization with no `AuthProvider`
  falls back to central providers; once it registers one or more, only those are
  accepted for that organization.

## Consequences

### Positive

- One authentication model for the whole platform; `vnext-service`s stay dumb
  about identity (front-proxy + TokenReview/SAR), which is what makes the
  aggregating surface (ADR-0006) viable.
- Per-tenant providers satisfied declaratively with the same
  admission/audit/list/watch machinery as every other resource.
- No second TTL to manage; the actual provider token is verified against its real
  issuer every request.

### Negative / risks

- vNext now performs JWT signature verification and JWKS management. Mitigated:
  verification is local and cached; JWKS rotation is handled per `AuthProvider`.
- `iss` uniqueness must be enforced at admission or the selection model is unsafe.
- Front-proxy header trust requires a mutually-authenticated channel between vNext
  and each `vnext-service`; misconfiguration would let a service trust spoofed
  identity headers. Mitigated by requiring the requestheader client identity and
  rejecting the headers from any other source.

### Follow-ups

- `AuthProvider` is added to the MVP resource list (PRD §33.1 amendment).
- Define the secret-reference mechanism for provider client secrets.
- Specify the `TokenReview`/`SubjectAccessReview` request/response contracts and
  the front-proxy header set + trust configuration.

## Alternatives considered

- **BFF verifies tokens (ADR-0002).** Superseded: incompatible with the
  aggregating single-surface model — authentication must be a property of the API
  server, not duplicated in every BFF.
- **Signed delegation envelope minted by vNext/BFF.** Rejected: introduces a
  second TTL layered on top of provider tokens with mismatched lifetimes, and
  re-implements what TokenReview/SAR already provide natively.
- **Single cluster-wide OIDC (vanilla k8s).** Rejected: the platform explicitly
  requires per-tenant providers; `iss`-keyed `AuthProvider` resources deliver
  that without forking the verification path.
