# PRD: vNext SaaS Control Plane

Status: Draft v0.4
Product Name: vNext
GitHub Organization: `vnextio`
Core Repository: `github.com/vnextio/vnext`
Type: SaaS Control Plane / API Machinery
Primary Reference Consumers: HR OS, Healthcare OS, Telry, Maritime SaaS, AI Agents / Call Center Platform
Secondary / Conditional Consumers: Dental Equipment Marketplace, Insurance Core

---

## 0. Amendments & Precedence (v0.4)

This document is the broad architecture narrative. Individual decisions are
recorded as ADRs in [`../adr/`](../adr/). **When documents conflict, precedence is
ADR > PRD > RFD.** RFD-0001 (`../rfd/0001-architecture-reference.md`) is
**superseded** and historical only.

v0.4 records the following decisions made after v0.3. Where a section below still
reads in the old way, **the amendment here and the cited ADR win**; inline edits
have been applied to the most load-bearing sections.

| # | Decision | Supersedes / amends | ADR |
| - | -------- | ------------------- | --- |
| 1 | **`Project` is killed.** One deployment per product makes it a constant, not a model concept. Hierarchy is `platform → organization → scope`. `ProjectID` is removed from `RequestContext` and from all `control_plane.*` tables. `${PROJECT}` survives only as a deployment/namespace name (an ops label). | §9, §23.4 | ADR-0007 |
| 2 | **Aggregating single API surface + unified discovery.** vNext is *the* API and discovery surface; consumers are backend-agnostic (`api-resources` model). Three serving mechanisms — built-in typed, registered dynamic (CRD-style), aggregated `vnext-service` — are indistinguishable to the client. | §22.4, §37.1 #5 | ADR-0006 |
| 3 | **Dynamic types are a day-one architectural invariant** (discovery + client assume them); implementation of the generic store/proxy is phased after the core slice. | §37.1 #4 | ADR-0006 |
| 4 | **Authentication is baked into the API server** (kube-apiserver model). `iss`-keyed central + per-tenant `AuthProvider`; validate `iss`+`aud`; unique `issuer`; verification local via cached JWKS. | §23.1, §23.3 | ADR-0004 (supersedes ADR-0002) |
| 5 | **Delegation via TokenReview/SubjectAccessReview + front-proxy identity injection.** The bespoke signed-delegation envelope is dropped; token refresh is the client/BFF's job, never vNext's. | §23.3 | ADR-0004 |
| 6 | **Strict-proxy reads** routed via discovery. Discovery cache always on; result cache **per-type opt-in** with TTL + `resource_events` invalidation; bounded pages. | §19 | ADR-0006 |
| 7 | **Deploy strictly on Kubernetes.** Two planes: control-plane *data* in Postgres; *operational* plane (config, secrets, service discovery, leader election) is k8s-native (ConfigMap/Secret/Service/`Lease`). The Postgres `control_plane.leases` table is retired. vNext resources are **not** CRDs. | §15, §21.2 | ADR-0007 |
| 8 | **Migrations are `vnext-service`-owned.** The control plane migrates only `control_plane.*`; each service self-migrates its own schema(s) on deploy with a schema-scoped DB role. No central orchestration; no cross-schema hard FKs. | §14, §35 | ADR-0007 |
| 9 | **`vnext-service`** is the umbrella term for backends that serve types and/or run controllers (replaces "module / domain API server / controller"). | §22, §29 | ADR-0007 |
| 10 | **Two reuse models, two layers** (see §5.7): platform = reused like a framework; domain = extended like k8s. Build vNext fresh, kernel as parts donor, strangler migration. | §3, §5, §35 | ADR-0005 |

Earlier resolved/open decisions also covered by ADRs: RBAC inheritance via
query-time ancestor check (ADR-0001); marketplace parked with the controller seam
kept open (ADR-0003).

---

## 1. Summary

vNext is a Kubernetes-inspired SaaS control plane for building multi-tenant SaaS products.

It provides reusable platform API machinery for:

* Tenants
* Scopes
* Principals/users
* Memberships
* RBAC
* Service accounts
* Signed service-to-service delegation
* Entitlements
* Quotas
* Admission/defaulting/validation
* Audit logs
* Resource definitions
* Resource registry
* Labels and annotations
* Public opaque IDs
* Generated clients
* Dynamic clients
* Controller/runtime interfaces
* API discovery
* Resource versions
* Watch/list semantics
* Status subresources
* Conditions
* Finalizers
* Events
* Leases

vNext is not a SaaS product by itself. It is the reusable control-plane foundation used by products such as HR OS, Healthcare OS, Telry, Maritime SaaS, and AI Agents / Call Center Platform.

Each SaaS product gets its own vNext deployment and product database in the first version. vNext should not begin as one shared global control plane for all products.

---

## 2. Naming and Repository Decision

### 2.1 Product / Platform Name

The platform name is:

```text
vNext
```

vNext is the internal and platform-facing name for the reusable SaaS control plane.

The name `kontrol-plane` should not be used as the main product name.

Reasons:

* Too Kubernetes-insider.
* Looks intentionally misspelled.
* Over-anchors the platform to Kubernetes.
* Sounds more like an infrastructure tool than SaaS API machinery.

### 2.2 GitHub Organization

Canonical GitHub organization:

```text
github.com/vnextio
```

Canonical core repository:

```text
github.com/vnextio/vnext
```

Initial repo structure:

```text
vnextio/
  vnext
```

Possible later splits:

```text
vnextio/
  vnext
  client-go
  vnextctl
  api
  docs
  examples
```

### 2.3 Product Repositories

Company/product repositories may remain under the company organization.

Example:

```text
edgescaleDev/
  hr-os
  healthcare-os
  telry
  maritime-saas
  ai-call-center
```

### 2.4 Contrib Modules

Reusable domain modules/extensions may live under a contrib organization.

Example:

```text
vnext-contrib/
  attendance
  shifts
  hierarchy
  appointments
  catalog
  payments
  prescriptions
  reports
  notifications
```

Rule:

```text
If breaking it breaks every vNext-based product, keep it in vnextio/vnext.

If it is a reusable product/domain extension, put it in vnext-contrib.

If it is product-specific, keep it in the product organization/repo.
```

---

## 3. Background

The existing kernel approach is useful for modular monolith development, but it has a long-term ceiling for strategic SaaS products.

The main limitation is:

```text
new module = compile new build = redeploy the product/control plane
```

vNext exists to move from:

```text
modular monolith / microkernel module framework
```

toward:

```text
Kubernetes-inspired SaaS API machinery
```

vNext borrows the useful ideas from Kubernetes:

* Stable resource identity
* API groups and versions
* Typed resources
* Metadata/spec/status discipline
* Admission checks
* RBAC
* Service accounts
* Controllers
* Generated clients
* Dynamic resources
* Resource versions
* Watch/list semantics
* Optimistic concurrency
* Status subresources
* Conditions
* Finalizers
* Events
* Auditability

But vNext does not copy Kubernetes blindly.

vNext does not:

* Use Kubernetes as the product API.
* Run product domains as Kubernetes CRDs.
* Expose raw Kubernetes APIs to product clients.
* Use etcd just because Kubernetes does.
* Treat the control plane as a generic database.
* Force all SaaS products into one global deployment.
* Bake product-specific modules into the control plane.

---

## 4. Product Vision

vNext should become the reusable SaaS control-plane foundation for building serious multi-tenant products.

Target products include:

* HR OS
* Healthcare OS
* Telry
* Maritime SaaS
* AI Agents / Call Center Platform
* Insurance Core
* Dental Procurement OS if the marketplace evolves into a platform

The long-term vision:

```text
Build SaaS products the way Kubernetes builds platforms:
stable API machinery, strong contracts, resource ownership, reconciliation, extensibility, and clear control-plane boundaries.
```

vNext should let product teams build SaaS platforms with consistent primitives while preserving product-specific domain behavior and user experience.

---

## 5. Core Product Principles

### 5.1 vNext Is a Control Plane, Not a Product

vNext owns generic platform primitives.

It must not own product-specific concepts such as:

* Employee
* Doctor
* Patient
* Appointment
* Attendance record
* Payroll run
* Prescription
* Medical note
* WhatsApp thread
* AI agent session
* Vessel
* Insurance policy
* Marketplace order

Those belong to domain API servers.

### 5.2 Products Own Product Experience

Product-facing clients should not consume raw platform APIs directly.

Each product should expose its own BFF APIs:

```text
/api/*
```

The product BFF owns:

* UX-friendly routes
* Aggregation
* Response shaping
* Permission-aware navigation
* Product workflows
* Backward-compatible response envelopes

### 5.3 Platform APIs Use Resource Discipline

vNext and domain API servers expose platform APIs:

```text
/apis/*
```

These APIs use resource-style contracts:

* `apiVersion`
* `kind`
* `metadata`
* `spec`
* `status` where useful
* `resourceVersion`
* `generation`
* `observedGeneration`
* paginated lists
* status subresources
* strict writes
* tolerant reads

### 5.4 Domain API Servers Own Business State

Domain API servers own:

* Domain tables
* Domain migrations
* Domain validation
* Domain workflows
* Domain events
* Domain reconciliation
* Domain projections
* Domain APIs

vNext must not become a dumping ground for product business tables.

### 5.5 Copy Kubernetes API Semantics, Not Kubernetes Storage Blindly

vNext should behave like an API server to clients, controllers, BFFs, and operators.

But vNext should use storage that fits SaaS control-plane data.

Decision:

```text
Postgres is the primary vNext control-plane datastore.

etcd is not used as the primary vNext datastore in v1.
```

### 5.6 Public IDs Are Opaque and Non-Secret

vNext supports public opaque IDs for support/debugging.

Examples:

```text
Tenant public ID
Scope public ID
Principal public ID
```

Products may label these differently.

For HR OS:

```text
Organization ID
Branch ID
User ID
```

These IDs are not secrets and must never replace authorization checks.

### 5.7 Two Reuse Models, Two Layers

vNext serves two intentions that pull in opposite directions, and it keeps them
from colliding by assigning each to its own layer:

```text
Platform layer  → reused like a framework (Laravel-style): shared, batteries-
                  included, stable. You get tenancy, RBAC, auth, audit,
                  discovery, resource machinery, and the client SDK once; every
                  product deploys an instance. Platform primitives are compiled
                  into vNext.

Domain layer    → extended like Kubernetes: register a type + run a
                  vnext-service/controller. Domain logic is NEVER compiled into
                  the control plane. Extending the platform requires no rebuild
                  of the core.
```

The kernel failed because it applied framework-style reuse to **both** layers —
domain modules were compiled into the framework binary, so "extend" meant
"rebuild." That collision is the rebuild-on-new-module ceiling vNext exists to
escape.

The dividing rule, when unsure whether something is platform or domain:

```text
Would breaking it break EVERY vNext product?
  yes -> platform primitive (compile into vNext)
  no  -> domain extension (lives in a vnext-service)
```

For this to hold in practice, **building a `vnext-service` must be as ergonomic as
a framework generator** (codegen from proto, a controller SDK, scaffolding, a
`vnextctl` generator). Excellent `vnext-service` developer experience is a
first-class goal, not late-phase polish — without it, teams take the shortcut of
compiling domain logic in, and the ceiling returns. See ADR-0005.

---

## 6. Product Fit Decisions

### 6.1 HR OS

Recommendation:

```text
Use vNext / Kubernetes-inspired API server.
```

Reason:

HR OS is a platform product with tenants, branches, users, roles, attendance, shifts, payroll, finance, reports, hierarchy, subscriptions, entitlements, and audit.

### 6.2 Healthcare OS

Recommendation:

```text
Strongly use vNext / Kubernetes-inspired API server.
```

Reason:

Healthcare OS has stronger boundary pressure than HR OS: patients, doctors, staff, corporate clients, partner pharmacies, appointments, prescriptions, payments, consent-like access, clinical data, quotas, and audit.

### 6.3 Maritime SaaS

Recommendation:

```text
Use vNext / Kubernetes-inspired API server.
```

Reason:

Maritime SaaS has natural multi-tenant and scope-heavy concepts: companies, fleets, vessels, ports, voyages, crew, certificates, inspections, maintenance, compliance, and document workflows.

### 6.4 AI Agents / Call Center Platform

Recommendation:

```text
Strongly use vNext / Kubernetes-inspired API server.
```

Reason:

AI agents/call-center is a control-plane product by nature. It manages agents, agent versions, SIP trunks, phone numbers, call flows, tools, knowledge bases, quotas, voice profiles, sessions, recordings, transcripts, runtime deployments, and provider integrations.

### 6.5 Dental Equipment Marketplace

Recommendation:

```text
Start with modular monolith / microkernel-style server.
Move toward vNext only if it becomes Dental Procurement OS.
```

Reason:

A simple marketplace is mostly transactional: sellers, buyers, products, inventory, orders, payments, shipping, reviews, quotes, and support. That does not require vNext initially.

Use vNext later if the product evolves into:

* Clinic procurement platform
* Supplier platform
* Multi-branch purchasing workflows
* Maintenance/warranty lifecycle platform
* Rental/subscription equipment platform
* Enterprise procurement approvals
* Clinic-specific entitlements

---

## 7. Goals

### 7.1 Platform Goals

1. Provide reusable SaaS control-plane primitives.
2. Support multi-tenant SaaS products.
3. Model organization and scope hierarchy.
4. Normalize principals, memberships, and role bindings.
5. Provide RBAC and authorization checks.
6. Provide service accounts and signed service-to-service delegation.
7. Provide feature grants, quotas, and effective entitlement snapshots.
8. Provide admission/defaulting/validation for writes.
9. Provide audit logs for security-sensitive actions.
10. Provide typed first-party resources.
11. Provide runtime/custom resources where appropriate.
12. Generate typed clients for first-party APIs.
13. Provide a dynamic client for runtime resources.
14. Support controller-style reconciliation.
15. Support labels and annotations.
16. Support public opaque IDs.
17. Support localized display names.
18. Support list/watch semantics for controllers.
19. Avoid coupling new modules to a single control-plane binary.
20. Keep v1 operationally practical.

### 7.2 Engineering Goals

1. Use Postgres as primary storage.
2. Use one physical database per SaaS product.
3. Use multiple PostgreSQL schemas per owner/domain.
4. Keep control-plane data in `control_plane.*`.
5. Keep product/domain data in domain-owned schemas.
6. Use strict table ownership even when services share a physical database.
7. Avoid hard cross-schema coupling where future database splits are likely.
8. Use typed fields for trusted business state.
9. Use labels for selection/grouping.
10. Use annotations for descriptive metadata/provenance/controller hints.
11. Support Kubernetes-like API-server semantics without requiring Kubernetes-like infrastructure complexity.

---

## 8. Non-Goals

vNext v1 will not:

1. Be a Kubernetes clone.
2. Expose Kubernetes APIs to SaaS clients.
3. Run product domains as Kubernetes CRDs.
4. Use etcd as the primary datastore.
5. Build a global shared control plane for all products.
6. Provide a public third-party marketplace.
7. Implement hot-loaded Go plugins.
8. Store all product data as generic JSON resources.
9. Replace every existing product backend immediately.
10. Force all domain APIs to use separate physical databases.
11. Force all domain APIs to become microservices from day one.
12. Own product-specific business workflows.
13. Own product-specific UI navigation.
14. Own product-specific mobile/web response envelopes.
15. Use labels or annotations as authorization boundaries.
16. Store secrets in annotations.
17. Store high-volume business data in generic resources.

---

## 9. Conceptual Model

Core vNext concepts:

> **`Project` was removed in v0.4 (ADR-0007).** One deployment per product makes
> it a constant, not a model concept. The hierarchy root is the `platform`
> tenant. `${PROJECT}` survives only as a deployment/namespace name (an ops
> label), never as a resource or a stored column.

```text
Tenant
Scope
Principal
Membership
Role
Permission
RoleBinding
ServiceAccount
FeatureGrant
Quota
EntitlementSnapshot
ResourceDefinition
APIService
AuditEvent
Event
Lease
```

Product mappings:

### HR OS

```text
Company       -> Organization tenant
Branch        -> Scope tenant
Employee      -> Principal + Membership + EmployeeProfile
Manager       -> RoleBinding
Attendance    -> FeatureGrant + Attendance domain API
Seat limit    -> Quota
```

### Healthcare OS

```text
Corporate client     -> Organization tenant
Partner pharmacy     -> Organization or partner tenant
Clinic/site/team     -> Scope tenant
Patient              -> Principal + PatientProfile
Medical staff        -> Principal + StaffProfile
Appointment booking  -> Appointments domain API
Consultation quota   -> Quota
```

### AI Call Center

```text
Customer organization -> Organization tenant
Project/workspace     -> Scope tenant
Agent                 -> Domain resource
SIP trunk             -> Domain resource
Phone number          -> Domain resource
Call session          -> Domain resource
Usage limit           -> Quota
```

---

## 10. Tenant and Scope Model

vNext supports hierarchical tenants/scopes.

Base model:

```text
platform tenant
  -> organization tenant
       -> scope tenant
```

vNext uses one `Tenant` resource for the hierarchy.

The hierarchy type is stored on:

```text
spec.kind
```

Initial tenant kinds:

```text
platform
organization
scope
```

Products may label or expose scope tenants with product vocabulary.

Examples:

```text
HR OS scope tenant         -> Branch
Healthcare scope tenant    -> Clinic / Team / Department
Maritime scope tenant      -> Fleet / Vessel / Port office
AI Call Center scope tenant -> Project / Workspace
```

Product-specific labels:

| vNext Concept       | HR OS                      | Healthcare OS              | Maritime                            | AI Call Center         |
| ------------------- | -------------------------- | -------------------------- | ----------------------------------- | ---------------------- |
| Organization tenant | Organization / Company     | Corporate client / Partner | Shipping company / Operator         | Customer organization  |
| Scope tenant        | Branch                     | Clinic / Team / Department | Fleet / Vessel / Port office        | Project / Workspace    |
| Principal           | User / Employee            | Patient / Doctor / Staff   | Crew / Staff / Agent                | Agent admin / Operator |
| RoleBinding         | Admin / Manager / Employee | Doctor / Support / Patient | Vessel manager / Compliance officer | Admin / Supervisor     |

Domain data should avoid ambiguous single `tenant_id` when both organization and scope matter.

Prefer:

```text
org_tenant_id
scope_tenant_id
```

or product-specific names:

```text
company_tenant_id
branch_tenant_id
partner_tenant_id
department_tenant_id
```

### 10.1 Core Resource Scope

Initial resolved scope model:

| Resource     | Scope                         |
| ------------ | ----------------------------- |
| Tenant       | Global                        |
| Principal    | Global                        |
| Role         | Global                        |
| Permission   | Global                        |
| RoleBinding  | Organization or scope/branch  |
| FeatureGrant | Organization or scope/branch  |
| Quota        | Organization or scope/branch  |

Other first-party resources define their exact scope as they enter the implementation slice.

---

## 11. Public IDs

### 11.1 Requirement

vNext provides opaque public identifiers for resources that products can safely show to end-users and support teams.

Public IDs are exposed as:

```text
metadata.publicID
```

Rules:

```text
metadata.name = stable resource name, caller-chosen or server-normalized
metadata.uid = immutable internal system UID
metadata.publicID = immutable opaque support/debug ID, server-generated, read-only
```

Initial public ID types:

```text
Tenant public ID
Scope public ID
Principal public ID
```

For HR OS, expose as:

```text
Organization ID
Branch ID
User ID
```

Avoid `Account ID` for HR OS because it may confuse end-users.

### 11.2 Format

Use Cloudflare-style opaque IDs.

The ID payload is a random UUIDv4 generated and validated through a UUID library.

Encoding rule:

```text
publicID = prefix + "_" + lowercase_dashless_uuid_v4
```

Example implementation rule:

```text
uuid.NewRandom() -> string -> remove "-" -> prefix
```

Recommended format:

```text
org_3f8a1c9e4b6241c6a84d02b997ad22f1
br_72acb42f5e2b4f9aa8d3190cc7e98134
usr_b13fd7d2e7be4f789d1ef8b7f5c231a9
```

Generic vNext alternatives:

```text
tnt_3f8a1c9e4b6241c6a84d02b997ad22f1
scp_72acb42f5e2b4f9aa8d3190cc7e98134
usr_b13fd7d2e7be4f789d1ef8b7f5c231a9
```

Product labels may differ, but public IDs remain opaque.

### 11.3 Security Rules

1. Public IDs are not secrets.
2. Public IDs must be non-sequential.
3. Public IDs must not encode customer names.
4. Public IDs must not encode plan information.
5. Public IDs must not expose environment names.
6. Possession of a public ID must never grant access.
7. All APIs must enforce authorization independently of public ID knowledge.

---

## 12. Names and Translations

### 12.1 Identity vs Display Text

Never translate:

```text
metadata.name
metadata.publicID
metadata.uid
resource identifiers
slugs/codes/references
```

Translate or localize:

```text
display_name
description
short_label
help_text
UI labels
product-facing names
```

### 12.2 Resource Shape

Example tenant resource:

```json
{
  "apiVersion": "core/v1",
  "kind": "Tenant",
  "metadata": {
    "name": "al-noor-hospital",
    "uid": "018f0c4d-2db8-7a8b-9f48-83e8c2cbb111",
    "publicID": "org_3f8a1c9e4b6241c6a84d02b997ad22f1"
  },
  "spec": {
    "kind": "organization",
    "displayName": "Al Noor Hospital",
    "localizedDisplayNames": {
      "ar-IQ": "مستشفى النور",
      "en": "Al Noor Hospital"
    }
  }
}
```

### 12.3 Personal Names

User/principal names should not be automatically machine-translated.

For users, localized names should be:

* user-provided,
* admin-provided,
* imported from source HR/identity data,
* or manually maintained.

Example:

```json
{
  "apiVersion": "iam/v1",
  "kind": "Principal",
  "metadata": {
    "name": "ahmed-obaidi",
    "uid": "018f0c4d-2db8-7a8b-9f48-83e8c2cbb222",
    "publicID": "usr_b13fd7d2e7be4f789d1ef8b7f5c231a9"
  },
  "spec": {
    "displayName": "Ahmed Obaidi",
    "localizedDisplayNames": {
      "ar-IQ": "أحمد العبيدي",
      "en": "Ahmed Obaidi"
    }
  }
}
```

### 12.4 Locale Fallback

BFFs should resolve display names using:

```text
1. user preferred locale
2. organization default locale
3. product default locale
4. resource display_name
5. metadata.publicID as last fallback
```

### 12.5 Storage

For MVP, use JSONB:

```sql
localized_display_names jsonb not null default '{}'
```

Later, if search/sorting/reporting by localized names becomes important, introduce:

```text
control_plane.resource_localizations
```

---

## 13. Labels and Annotations

### 13.1 Core Rule

```text
Labels are for selection, grouping, filtering, and automation.

Annotations are for non-query-critical metadata, controller hints, provenance, and external references.
```

Labels and annotations must not become the business schema.

### 13.2 Labels

Labels are small, indexed, queryable key/value metadata.

Good label use cases:

```text
product ownership
environment
module ownership
controller ownership
migration state
geographic grouping
resource grouping
operator selection
```

Examples:

```json
{
  "metadata": {
    "labels": {
      "vnext.io/product": "hr-os",
      "vnext.io/environment": "prod",
      "vnext.io/managed-by": "hr-bff",
      "vnext.io/module": "attendance",
      "vnext.io/imported-from": "tawajud",
      "hr.edgescale.dev/country": "iq"
    }
  }
}
```

### 13.3 Label Selectors

vNext should support label selectors.

Examples:

```text
GET /apis/core/v1/tenants?labelSelector=vnext.io/product=hr-os

GET /apis/core/v1/tenants?labelSelector=vnext.io/environment=prod,vnext.io/module=attendance
```

Client example:

```go
client.CoreV1().
    Tenants().
    List(ctx, ListOptions{
        LabelSelector: "vnext.io/product=hr-os,vnext.io/environment=prod",
    })
```

### 13.4 Annotations

Annotations are larger, usually non-indexed descriptive metadata.

Good annotation use cases:

```text
migration provenance
legacy IDs
configuration origin
external provider references
controller hints
support notes
manual operator notes
```

Examples:

```json
{
  "metadata": {
    "annotations": {
      "vnext.io/description": "Migrated from Tawajud production database.",
      "vnext.io/config-origin": "migration",
      "vnext.io/legacy-id": "companies/1242",
      "vnext.io/last-imported-at": "2026-06-26T12:00:00Z",
      "support.edgescale.dev/customer-ticket": "SUP-18291"
    }
  }
}
```

### 13.5 What Not to Put in Labels or Annotations

Do not store secrets:

```text
API keys
tokens
passwords
private keys
OAuth refresh tokens
database URLs
SIP passwords
provider credentials
```

Do not store high-volume data:

```text
attendance events
call transcripts
chat messages
payment ledger entries
clinical notes
analytics facts
```

Do not store authoritative business state:

```text
employee salary
appointment status
payment status
prescription status
attendance status
subscription state
seat usage count
```

### 13.6 Labels Are Not Authorization

Bad:

```text
if resource.labels["branch"] == user.branch:
    allow
```

Good:

```text
Check membership + role binding + scope + permission + entitlement.
```

### 13.7 Reserved Namespaces

System-managed:

```text
vnext.io/*
core.vnext.io/*
iam.vnext.io/*
entitlements.vnext.io/*
```

Product-managed:

```text
hr.edgescale.dev/*
health.edgescale.dev/*
telry.edgescale.dev/*
ai.edgescale.dev/*
maritime.edgescale.dev/*
```

Controller-managed:

```text
attendance.edgescale.dev/*
reports.edgescale.dev/*
networking.edgescale.dev/*
```

User/customer-defined:

```text
custom/*
customer.edgescale.dev/*
```

### 13.8 Admission Rules

vNext admission must enforce:

1. Valid label/annotation key format.
2. Valid value format.
3. Maximum label count.
4. Maximum annotation size.
5. Reserved prefixes can only be written by authorized callers.
6. Immutable labels cannot be changed after creation.
7. Enum labels accept only allowed values.
8. Detectable secrets should be rejected where possible.

---

## 14. Storage Decision

### 14.1 Primary Storage

Decision:

```text
Postgres is the primary datastore for vNext control-plane data.
```

Do not use etcd as the primary datastore in v1.

Reason:

vNext control-plane data is SaaS platform data, not cluster state.

It needs:

```text
relational constraints
transactions
unique indexes
joins
debug/support queries
audit queries
migrations
JSONB where useful
backup/restore familiarity
multi-tenant filtering
```

### 14.2 Product Database Strategy

Decision:

```text
One physical database per SaaS product.
Multiple schemas per owner/domain.
```

Example HR OS:

```text
hr_os_db
  control_plane.*
  subscription.*
  hr.*
  attendance.*
  shifts.*
  payroll.*
  finance.*
  reports.*
  notifications.*
```

Example Healthcare OS:

```text
healthcare_os_db
  control_plane.*
  subscription.*
  healthcare.*
  catalog.*
  appointments.*
  payments.*
  prescriptions.*
  threads.*
  partners.*
  corporate.*
  analytics.*
```

### 14.3 Schema Ownership

Rule:

```text
Physical database boundary = operational boundary.
Schema boundary = ownership boundary.
Table boundary = implementation detail.
```

Control plane owns:

```text
control_plane.*
```

Domain modules own:

```text
attendance.*
appointments.*
payments.*
threads.*
payroll.*
finance.*
reports.*
```

### 14.4 Cross-Schema References

Within a schema, foreign keys are allowed and encouraged.

Across schemas, use caution.

Preferred rule:

```text
Use UUID/logical references across schemas.
Avoid hard cross-schema foreign keys where future database splitting is likely.
Validate through APIs/admission/reconciliation.
```

Example attendance table may store:

```text
org_tenant_id
scope_tenant_id
principal_id
membership_id
```

but should not necessarily enforce all of these through hard FKs to `control_plane.*`.

### 14.5 When to Split Physical Databases

Split a module into its own database only when one of these becomes true:

1. High write volume threatens the product DB.
2. Different backup/retention policy is required.
3. Different security/compliance boundary is required.
4. Independent scaling is required.
5. Independent restore/migration is required.
6. The module becomes reusable across products.
7. The module team needs operational autonomy.
8. Reporting workload hurts transactional workload.

Likely future split candidates:

```text
notifications
analytics/reports
chat/threads
call sessions/transcripts
audit archive
finance ledger
clinical records
recording/transcript metadata
```

---

## 15. Control-Plane Data Model

Initial control-plane schema:

```text
control_plane.tenants
control_plane.principals
control_plane.memberships
control_plane.roles
control_plane.permissions
control_plane.role_bindings
control_plane.service_accounts
control_plane.feature_grants
control_plane.quotas
control_plane.entitlement_snapshots
control_plane.api_services
control_plane.audit_events
control_plane.resource_events
control_plane.outbox_events
```

> **v0.4 amendments (ADR-0007):**
> - `control_plane.leases` is **retired**. Controller leader election uses
>   Kubernetes-native `coordination.k8s.io/Lease` (vNext deploys strictly on k8s).
> - No `project_id` column on any `control_plane.*` table (`Project` is killed).
> - `resource_events` and `outbox_events` are distinct by purpose:
>   `resource_events` is the **watch/informer change feed** (the source for
>   list/watch, §16) consumed internally; `outbox_events` is for **external event
>   publishing** (e.g. NATS), optional and post-MVP. A single write may append to
>   both within its transaction, but they are not the same table.

`ResourceDefinition` storage is delayed until runtime/custom resources enter the implementation scope.

Storage identity mapping:

```text
id        -> metadata.uid
name      -> metadata.name
public_id -> metadata.publicID
deleted_at -> metadata.deletionTimestamp
```

For tenants, `kind` is a queryable storage copy of `spec.kind`.

### 15.1 Soft Deletes

Use soft deletes when the resource or domain row represents mutable business/control-plane state where auditability, support recovery, or delayed cleanup matters.

Storage convention:

```text
deleted_at timestamptz null
```

API convention:

```text
metadata.deletionTimestamp
```

Domain modules that use GORM should use GORM's soft-delete convention for applicable mutable records, usually through `gorm.DeletedAt` or an equivalent `deleted_at` field.

Soft-delete rules:

1. Normal reads and lists exclude rows where `deleted_at is not null`.
2. Raw SQL must explicitly filter `deleted_at is null` when it wants active rows.
3. Resource events still emit a `DELETED` event so controller caches remove the object.
4. The deleted row remains available for audit/debug, retention, restore, or later purge flows.
5. Unique indexes must be chosen deliberately: full unique indexes if names must never be reused, or active-row partial/composite indexes if reuse after delete is allowed.

Do not use soft deletes for append-only records whose correctness depends on immutability.

Examples:

```text
audit_events
resource_events
outbox_events
ledger entries
usage facts
high-volume immutable event streams
```

Example tenant table:

```sql
control_plane.tenants (
    id uuid primary key,
    name text not null unique,
    public_id text not null unique,
    kind text not null,
    parent_id uuid null,
    display_name text not null,
    localized_display_names jsonb not null default '{}',
    labels jsonb not null default '{}',
    annotations jsonb not null default '{}',
    resource_version bigint not null,
    generation bigint not null default 1,
    status text not null,
    created_at timestamptz not null,
    updated_at timestamptz not null,
    deleted_at timestamptz null
);
```

Example principal table:

```sql
control_plane.principals (
    id uuid primary key,
    name text not null unique,
    public_id text not null unique,
    display_name text not null,
    localized_display_names jsonb not null default '{}',
    primary_email text null,
    primary_phone text null,
    labels jsonb not null default '{}',
    annotations jsonb not null default '{}',
    resource_version bigint not null,
    generation bigint not null default 1,
    status text not null,
    created_at timestamptz not null,
    updated_at timestamptz not null,
    deleted_at timestamptz null
);
```

---

## 16. Resource Version and Watch Storage

### 16.1 Resource Version

vNext should use a global monotonically increasing resource version sequence.

Example:

```sql
create sequence control_plane.resource_version_seq;
```

Every control-plane write receives:

```text
nextval('control_plane.resource_version_seq')
```

Resource versions are used for:

* watch resume
* optimistic concurrency
* controller cache correctness
* safe updates
* audit correlation

### 16.2 Durable Resource Events

Do not rely only on PostgreSQL `LISTEN/NOTIFY`.

Use a durable resource event table:

```sql
control_plane.resource_events (
    id bigserial primary key,
    resource_version bigint not null,
    event_type text not null,
    group_name text not null,
    version text not null,
    kind text not null,
    org_tenant_id uuid null,
    scope_tenant_id uuid null,
    name text not null,
    uid uuid not null,
    object jsonb null,
    created_at timestamptz not null
);
```

> **v0.4 amendment:** the original `namespace text null` column is removed. vNext
> has **no Namespace concept** (§10); scope is carried by `org_tenant_id` /
> `scope_tenant_id`, consistent with how domain resources are scoped. The
> `namespace` term was a Kubernetes habit and does not apply here.

Watch flow:

```text
1. Client lists resources.
2. Response includes latest resourceVersion.
3. Client starts watch from resourceVersion.
4. API server streams matching rows from resource_events.
5. If resourceVersion is too old, return expired/gone and force relist.
```

---

## 17. API-Server Semantics

vNext API server should behave like the authoritative API surface for platform resources.

Clients, controllers, BFFs, CLIs, and operators must depend on the vNext API, not on vNext database tables.

Required API-server semantics:

```text
resource discovery
typed REST APIs
list APIs with pagination
watch/event APIs
resourceVersion
generation
observedGeneration
optimistic concurrency
status subresources
conditions
finalizers
owner references
events
leases
admission/defaulting/validation
RBAC authorization
entitlement/quota checks
service account auth
signed delegation verification
audit logging
resource registry
controller-friendly clients
```

---

## 18. API Discovery

vNext should expose API discovery endpoints:

```text
GET /apis
GET /apis/core/v1
GET /apis/iam/v1
GET /apis/entitlements/v1
GET /apis/apiregistration.vnext/v1
GET /apis/attendance.hr/v1
```

Discovery must expose:

```text
groups
versions
resources
verbs
scope
status subresource support
watch support
short names if applicable
```

---

## 19. List and Watch

Controllers should follow the standard list/watch pattern:

```text
1. Initial list.
2. Store resourceVersion.
3. Watch from that resourceVersion.
4. Re-list if watch expires or falls behind.
5. Reconcile idempotently.
```

Example API:

```text
GET /apis/core/v1/tenants?limit=100
GET /apis/core/v1/tenants/events?resourceVersion=12345
```

v1 decision:

```text
list + resourceVersion + durable resource_events polling
```

The v1 API should preserve Kubernetes-like resume semantics even if transport is polling instead of streaming.

Longer-term target:

```text
list + watch + informer/cache semantics
GET /apis/core/v1/tenants?watch=true&resourceVersion=12345
```

---

## 20. Generation, Status, Conditions, and Finalizers

### 20.1 Generation

Use `generation` when a resource has desired state.

Rule:

```text
spec changes increment generation.
status updates do not increment generation.
```

### 20.2 Observed Generation

Controllers should write:

```text
status.observedGeneration
```

This indicates whether the controller has observed the latest desired state.

### 20.3 Status Subresource

Controllers update status separately from spec:

```text
PUT /apis/{group}/{version}/{resource}/{name}/status
```

Rule:

```text
BFF/user/admin owns spec.
Controller owns status.
```

### 20.4 Conditions

Use standard conditions for asynchronous state.

Example:

```yaml
status:
  conditions:
    - type: Ready
      status: "True"
      reason: Synced
      message: Resource is ready.
      lastTransitionTime: "2026-06-26T12:00:00Z"
```

### 20.5 Finalizers

Use finalizers for safe deletion of external state.

Example:

```yaml
metadata:
  finalizers:
    - networking.vnext.io/dns-cleanup
```

Deletion flow:

```text
1. User deletes resource.
2. If finalizers exist, vNext sets metadata.deletionTimestamp and keeps the resource visible to controllers.
3. Controller sees resource is deleting.
4. Controller cleans external state.
5. Controller removes finalizer.
6. vNext soft-deletes the resource by setting deleted_at, hides it from normal reads/lists, and emits a DELETED event.
```

---

## 21. Events and Leases

### 21.1 Events

Controllers and services should emit events.

Examples:

```text
Normal  Synced
Warning ProviderError
Normal  CertificateRenewed
Warning QuotaExceeded
Normal  AttendanceSummaryRecalculated
```

Events should be queryable by:

```text
tenant
scope
resource kind
resource name
request ID
controller name
```

### 21.2 Leases

> **v0.4 amendment (ADR-0007):** vNext deploys strictly on Kubernetes, so
> controller leader election uses Kubernetes-native `coordination.k8s.io/Lease`
> and its battle-tested leader-election machinery. The previously proposed
> `control_plane.leases` Postgres table is **retired** — there is no need to
> hand-roll leases when the substrate provides them.
>
> This is part of the operational/control-plane split: *operational* concerns
> (leader election, config, secrets, service discovery) are k8s-native;
> *control-plane data* stays in Postgres.

A `vnext-service` that runs controllers participates in standard Kubernetes
leader election so that only one replica reconciles at a time. No vNext-specific
lease resource is exposed on the API.

---

## 22. Resource Definitions and API Services

### 22.1 First-Party Resources

First-party/core resources should be defined using protobuf plus Buf.

Generated JSON Schema from the protobuf contracts is the primary REST/admission validation contract.

Generate:

```text
Go types
Go typed clients
OpenAPI
JSON Schema
validation stubs
admission stubs
```

First-party resources include:

```text
Tenant
Principal
Membership
Role
Permission
RoleBinding
ServiceAccount
FeatureGrant
Quota
EntitlementSnapshot
APIService
AuditEvent
Event
Lease
```

`ResourceDefinition` is a post-MVP first-party resource. It is introduced when runtime/custom resources are implemented.

### 22.2 Runtime/Custom Resources

Runtime/custom resources are post-MVP.

Runtime/custom resources use:

```text
ResourceDefinition + JSON Schema
```

Use for:

```text
configuration
desired state
controller-managed state
custom workflows
custom report definitions
low/medium-volume custom resources
```

Do not use for:

```text
attendance events
payment ledgers
chat messages
clinical notes
payroll runs
analytics facts
call transcripts
```

### 22.3 Domain API Server Registration

Domain API servers should register themselves with vNext using `APIService`.

Example:

```yaml
apiVersion: apiregistration.vnext/v1
kind: APIService
metadata:
  name: attendance.hr.v1
spec:
  group: attendance.hr
  version: v1
  service:
    name: attendance-api
    namespace: hr-os
  resources:
    - attendance-records
    - clock-events
    - daily-summaries
```

### 22.4 Aggregation vs Direct Calls

Two supported models:

#### Model A: vNext as API gateway/proxy

```text
client
  -> vnext-api-server
      -> registered domain API server
```

vNext handles:

```text
authn
authz
admission envelope
resource discovery
audit envelope
routing
```

Domain API handles:

```text
business validation
domain persistence
domain status
domain events
```

#### Model B: Domain APIs called directly

```text
BFF/client
  -> attendance-api
```

The domain API must implement the vNext platform API contract strictly.

```text
same auth/delegation model
same resource shape
same status conventions
same list/watch behavior where needed
same audit hooks
same generated client conventions
same error model
same metadata/spec/status discipline
```

Domain APIs must not invent incompatible envelopes or authorization shortcuts on `/apis/*`.

Domain API compatibility should be enforced with shared contract packages, generated clients, middleware, and conformance tests.

Recommended approach:

> **v0.4 amendment (ADR-0006, supersedes §37.1 #5).** The earlier "BFFs call
> domain APIs directly" recommendation is **reversed**. vNext is the single
> aggregating API surface; **consumers must not know which backend serves a
> type** — they discover it the way `kubectl api-resources` does, and reach it
> through one client. Direct-to-domain calls leak backend topology to clients and
> are not used.

```text
Clients use ONE vNext endpoint + ONE discovery surface.
vNext aggregates: built-in typed | registered dynamic (CRD-style) | aggregated vnext-service.
Aggregated types are proxied to their owning vnext-service (Model A),
  with front-proxy identity injection (§23.3).
```

Reads are strict-proxy through vNext, routed via the discovery cache; result
caching is per-type opt-in with a TTL (ADR-0006). High-volume domain data still
lives in `vnext-service`s (typed tables), never in the generic store (§5.4).
Implementation of the generic store (dynamic types) and the proxy is phased after
the core slice, but the discovery + client contract assumes them from day one.

---

## 23. Identity, Auth, and Delegation

### 23.1 Baked-in Authentication

> **v0.4 amendment (ADR-0004, supersedes ADR-0002).** The earlier stance — "vNext
> is not the external identity provider; BFFs verify tokens" — is **reversed**.
> Because vNext is the single aggregating API surface (ADR-0006), authentication
> must be a property of the API server itself, exactly as kube-apiserver
> authenticates every request.

vNext is **not** an identity provider (it does not issue end-user credentials or
run login UIs), but it **does verify** the token presented on every request.

**Authentication chain (first-to-accept wins):**

```text
1. ServiceAccount tokens  -> vNext-issued, vNext-signed JWTs (machine-to-machine)
2. Central / platform OIDC -> one configured external provider (k8s --oidc-* analog)
3. Per-tenant OIDC         -> each an AuthProvider resource (extension beyond vanilla k8s)
```

vNext still verifies tokens from external providers such as Keycloak, Zitadel,
Firebase Auth, Auth0, or any custom OIDC provider — the difference is that
verification happens **in the API server**, not in each BFF.

**Provider selection — by issuer (`iss`):**

```text
token.iss -> selects the AuthProvider whose JWKS verifies the signature
assert: signature (cached JWKS), then iss, then aud, then exp
```

- `AuthProvider.spec.issuer` is **unique**; admission rejects a duplicate issuer.
- `aud` must be validated, not just `iss`: `iss` selects the keys, `aud` proves
  the token was minted for this deployment.
- Verification is local once JWKS is cached — no network call on the hot path.

A BFF (or any consumer) forwards the user's token to vNext; vNext authenticates it
and resolves the principal/membership/tenant/roles context (§23.4). **Token
refresh is the client/BFF's responsibility, never vNext's** — vNext holds no
refresh tokens and no sessions; it only verifies the token presented per request.

See `AuthProvider` (`iam/v1`) in §33.1 and ADR-0004 for the resource shape.

### 23.2 Service Accounts

vNext supports service accounts for:

```text
BFFs
domain API servers
controllers
workers
operators
CLIs
```

### 23.3 Delegation Across the Aggregation Boundary

> **v0.4 amendment (ADR-0004).** The bespoke "BFF signs a short-lived delegation
> token" model is **dropped**. It would layer a second TTL on top of provider
> tokens with mismatched lifetimes and re-implements what Kubernetes already
> provides natively. Two k8s-native mechanisms replace it.

Because vNext authenticates every request (§23.1) and is the aggregating proxy
(ADR-0006), an aggregated `vnext-service` does not re-authenticate end users:

**1. Front-proxy identity injection (primary).** vNext authenticates once, then
forwards to the owning `vnext-service` with trusted identity headers (the
Kubernetes `--requestheader-*` / `X-Remote-User` model) over a
mutually-authenticated channel. The `vnext-service` trusts those headers **only**
from vNext, never from arbitrary clients.

**2. TokenReview / SubjectAccessReview (callback).** A `vnext-service` reached
directly, or one that wants to verify independently, calls back to vNext's
`TokenReview` / `SubjectAccessReview` endpoints. Same authority, opposite
direction.

```text
Request flow (aggregated path):
1. Client presents the user's token to vNext.
2. vNext authenticates it (iss -> AuthProvider, validate iss+aud+exp+sig).
3. vNext authorizes (RBAC, §24) and resolves principal/membership/tenant/roles.
4. vNext proxies to the owning vnext-service with trusted identity headers.
5. The vnext-service performs its own admission/business checks, then writes.
```

Downstream services trust only verified vNext front-proxy identity or a vNext
`TokenReview`/`SAR` result — never arbitrary client-provided context headers.
Authentication results may be cached with a short TTL bounded by token `exp`.

### 23.4 Request Context

Normalized request context:

```go
type RequestContext struct {
    OrgTenantID      string
    ScopeTenantID    string
    ScopeKind        string
    PrincipalID      string
    MembershipID     string
    AuthProviderID   string
    AuthSubject      string
    Roles            []string
    Permissions      []string
    EnabledFeatures  []string
}
```

> **v0.4 amendment (ADR-0007):** `ProjectID` is removed — `Project` is killed.
> One deployment serves one product, so the project is an ambient deployment
> constant, not a per-request field. The RFD-0001 `RequestContext` (with
> `ProjectID`, `SecurityProfile`, `ModulesEnabled`) is superseded by this shape;
> `ModulesEnabled` is covered by `EnabledFeatures`.

Domain services should be testable with this context directly.

---

## 24. RBAC

vNext owns:

```text
roles
permissions
role bindings
membership-based authorization
service account authorization
delegation verification
```

Every platform or domain write must check:

1. Actor identity.
2. Tenant membership.
3. Scope access.
4. Permission.
5. Feature entitlement where relevant.
6. Admission rules.

Possession of a resource ID must never imply access.

---

## 25. Entitlements and Quotas

### 25.1 Responsibility Split

Subscription module owns commercial state:

```text
plans
pricing
subscriptions
trials
billing periods
invoices
payments
enterprise contract rules
```

vNext owns runtime enforcement state:

```text
feature grants
quotas
effective entitlement snapshots
admission checks
```

### 25.2 Feature Grants

Examples:

```text
attendance enabled
payroll enabled
reports enabled
online consultation enabled
WhatsApp channel enabled
AI voice agents enabled
maritime compliance enabled
```

### 25.3 Quotas

Examples:

```text
active users <= 50
branches <= 10
monthly consultations <= 500
WhatsApp conversations <= 10000
AI call minutes <= 50000
storage <= 100GB
```

### 25.4 Entitlement Snapshot

vNext maintains an effective entitlement snapshot so domain services do not depend directly on billing availability.

Example:

```text
tenant hospital-a:
  attendance enabled = true
  reports enabled = true
  active_member limit = 50
```

---

## 26. Admission, Defaulting, and Validation

All platform writes pass through:

```text
authentication
authorization
defaulting
schema validation
business validation
entitlement checks
quota checks
audit decision
persistence
event emission
```

Server-side defaulting examples:

```text
default tenant status
default membership status
default role binding scope
default resource labels
default feature grant status
```

Validation rules:

```text
strict before writes
tolerant on reads
```

Reads should tolerate old stored data so caches, operators, and migrations are not broken by legacy shapes.

---

## 27. API Surfaces

### 27.1 Product API

Owned by product BFFs.

Prefix:

```text
/api/*
```

Used by:

```text
web clients
mobile clients
external product consumers where applicable
```

Characteristics:

```text
product-friendly response envelope
aggregation
permission-aware navigation
workflow-specific routes
backward compatibility
```

### 27.2 Platform API

Owned by vNext and domain API servers.

Prefix:

```text
/apis/*
```

Used by:

```text
BFFs
controllers
workers
operators
generated clients
dynamic clients
internal tools
```

Characteristics:

```text
resource-style responses
typed resources
paginated lists
server-side validation
resource versions
status subresources
watch support where appropriate
auditability
```

### 27.3 API Group Naming

Initial convention:

```text
/apis/{group}/{version}/{resource}
```

Examples:

```text
/apis/core/v1/tenants
/apis/iam/v1/principals
/apis/iam/v1/memberships
/apis/iam/v1/rolebindings
/apis/entitlements/v1/featuregrants
/apis/entitlements/v1/quotas
/apis/apiregistration.vnext/v1/apiservices
```

Domain examples:

```text
/apis/attendance.hr/v1/attendance-records
/apis/appointments.health/v1/appointments
/apis/threads.telry/v1/threads
/apis/agents.ai/v1/agents
/apis/maritime.v1/vessels
```

---

## 28. Client Model

### 28.1 Typed Clients

vNext provides generated typed clients for first-party APIs.

Example:

```go
client.CoreV1().Tenants().Get(ctx, name, opts)
client.IAMV1().Principals().Get(ctx, name, opts)
client.IAMV1().Memberships(namespace).List(ctx, opts)
client.EntitlementsV1().FeatureGrants(namespace).List(ctx, opts)
```

Domain clients:

```go
client.AttendanceV1().Records(scope).Get(ctx, name, opts)
client.AppointmentsV1().Appointments(scope).Create(ctx, appointment, opts)
client.ThreadsV1().Threads(scope).List(ctx, opts)
client.AgentsV1().Agents(scope).Get(ctx, name, opts)
```

### 28.2 Dynamic Client

Dynamic clients are post-MVP.

vNext provides a dynamic client for runtime/custom resources.

Example:

```go
client.Dynamic().
    Resource(gvr).
    Namespace(scope).
    Get(ctx, name, opts)
```

Dynamic client should not replace typed first-party clients.

Avoid relying on:

```go
sdk.Client[any]
```

as the primary client pattern.

---

## 29. Controller Runtime

vNext supports controller-style services.

Controllers watch resources/events and reconcile desired/observed state.

Examples:

```text
EntitlementSnapshotController
MembershipStatusController
AuditProjectionController
AttendanceSummaryController
PaymentStatusController
AppointmentReminderController
AgentDeploymentController
PhoneNumberController
KnowledgeIndexController
CertificateController
DomainBindingController
```

Controllers own:

```text
status
conditions
projections
derived state
external reconciliation
```

Controllers should not silently mutate caller-owned `spec`.

---

## 30. Example Controller Patterns

### 30.1 cert-manager-Like Pattern

vNext equivalent resources:

```text
CustomDomain
CertificateIssuer
CertificateRequest
SecretRef
```

Controller behavior:

```text
watch CustomDomain
validate DNS ownership
request/renew certificate
store certificate reference
update status.conditions
emit events
```

### 30.2 external-dns-Like Pattern

vNext equivalent resources:

```text
DomainBinding
DNSProvider
ProductEndpoint
```

Controller behavior:

```text
watch DomainBinding
compute desired DNS records
compare provider state
apply DNS changes
write status.conditions
use ownership markers
emit events
```

### 30.3 AI Agent Deployment Pattern

vNext equivalent resources:

```text
Agent
AgentVersion
AgentDeployment
VoiceProfile
SIPTrunk
PhoneNumber
KnowledgeBase
```

Controller behavior:

```text
watch AgentDeployment
prepare runtime config
provision LiveKit/SIP/provider state
sync knowledge index
update deployment status
emit events
```

---

## 31. Security Requirements

### 31.1 Object-Level Authorization

Every object access must be authorized.

This applies even if the caller knows:

```text
tenant ID
scope ID
principal ID
resource name
public ID
```

### 31.2 ID Exposure

vNext may expose opaque public IDs.

vNext must not expose sequential internal database IDs to end-users.

### 31.3 Service Trust

Downstream services trust only:

```text
verified service identity
verified signed delegation context
```

They must not trust client-provided context headers.

### 31.4 Cross-Tenant Safety

Default behavior is deny.

A user in Tenant A must not access Tenant B unless explicitly authorized.

A manager in Scope A must not access Scope B unless explicitly authorized.

### 31.5 Least Privilege

Service accounts must have scoped permissions.

BFF service accounts should not have unrestricted domain write access unless needed.

Controllers should have permissions only for the resources they reconcile.

---

## 32. Audit and Observability

### 32.1 Audit Log

Audit all sensitive actions:

```text
tenant create/update/delete
principal create/update/delete
membership create/update/delete
role binding create/update/delete
feature grant create/update/delete
quota create/update/delete
service account create/update/delete
APIService registration changes
domain-sensitive writes where delegated to vNext audit
resource definition create/update/delete once ResourceDefinition is enabled
```

Audit event fields:

```text
id
timestamp
actor_principal_id
actor_service_account_id
organization_tenant_id
scope_tenant_id
action
resource_group
resource_version
resource_kind
resource_name
resource_uid
request_id
decision
reason
source_ip
user_agent
```

### 32.2 Metrics

Required metrics:

```text
request count
request latency
admission latency
authorization deny count
entitlement deny count
quota deny count
audit write latency
watch reconnect count
watch expired count
controller reconcile count
controller reconcile errors
outbox lag
event processing lag
```

### 32.3 Tracing

Every request should propagate:

```text
request_id
trace_id
actor
organization tenant
scope tenant
service account
```

---

## 33. vNext MVP Scope

### 33.1 Core MVP Resources

Build:

```text
core.v1.Tenant
iam.v1.Principal
iam.v1.Membership
iam.v1.Role
iam.v1.Permission
iam.v1.RoleBinding
iam.v1.ServiceAccount
iam.v1.AuthProvider
entitlements.v1.FeatureGrant
entitlements.v1.Quota
entitlements.v1.EntitlementSnapshot
core.v1.AuditEvent
apiregistration.vnext/v1.APIService
```

> **v0.4 amendments:**
> - **`iam.v1.AuthProvider` added** (ADR-0004): authentication is baked in, so the
>   provider config the authn chain needs is a first-party MVP resource.
> - **`coordination.vnext/v1.Lease` removed** (ADR-0007): leader election uses
>   Kubernetes-native `coordination.k8s.io/Lease`; vNext exposes no lease
>   resource.

Explicitly delayed from MVP (implementation only — the discovery/client contract
assumes dynamic types from day one, per ADR-0006):

```text
core.v1.ResourceDefinition
runtime/custom resources (CRD-style generic storage)
dynamic client
generic resource table
aggregation proxy to external vnext-services
```

### 33.2 Core MVP Services

Build:

```text
vNext API server
authorization service
admission service
entitlement service
audit service
resource registry
APIService registry
service-account/delegation verifier
generated Go client
basic controller manager
resource event store
```

### 33.3 Core MVP APIs

Build:

```text
Tenant CRUD/List
Principal CRUD/List
Membership CRUD/List
Role CRUD/List
RoleBinding CRUD/List
ServiceAccount CRUD/List
FeatureGrant CRUD/List
Quota CRUD/List
EntitlementSnapshot read
AuditEvent write/read
APIService CRUD/List
Lease CRUD/Update
basic list/watch for core resources
```

### 33.4 Storage MVP

Build:

```text
Postgres control_plane schema
resource_version sequence
resource_events table
audit_events table
labels JSONB
annotations JSONB
localized_display_names JSONB
GIN indexes for labels
durable event polling for v1 watch semantics
```

### 33.5 First Product Proof

Use HR OS Attendance as the first reference consumer.

Success case:

```text
Organization has Attendance enabled and active user limit = 50.

User logs in through HR BFF.

BFF resolves principal, organization, branch/scope, membership, roles, permissions, and enabled features.

BFF calls Attendance API with signed request context.

Attendance API verifies authorization and entitlement state through vNext.

Attendance API creates a clock event and attendance record.

Audit trail records the action.
```

---

## 34. Acceptance Criteria

### 34.1 Tenant Acceptance

Given an organization tenant exists, vNext can:

* Return it by public ID.
* Return it by internal ID for service calls.
* List child scopes.
* Filter it by labels.
* Return localized display name.
* Deny access to unauthorized principals.
* Audit updates.

### 34.2 Principal Acceptance

Given a user logs in through a product BFF, vNext can resolve or create:

* Principal
* Membership
* Role bindings
* Effective permissions
* Enabled features

### 34.3 RBAC Acceptance

Given a principal has a role binding in Scope A, they can perform allowed actions in Scope A but not in Scope B.

### 34.4 Entitlement Acceptance

Given a tenant does not have a feature enabled, domain APIs can deny actions depending on that feature through vNext entitlement checks.

### 34.5 Quota Acceptance

Given a tenant has an active-user quota of 50, vNext admission denies creation or activation of user number 51.

### 34.6 Public ID Acceptance

Given a support agent receives a public tenant/scope/principal ID, the system can use it for lookup and debugging without exposing internal sequential database IDs.

### 34.7 Labels Acceptance

Given tenants have labels, clients can list tenants using label selectors.

Reserved label prefixes cannot be changed by unauthorized callers.

### 34.8 Annotations Acceptance

Given a migrated resource has annotations, support and migration tools can read legacy provenance metadata.

Annotations are not used for authorization or critical business state.

### 34.9 Localization Acceptance

Given a user preferred locale exists and the resource has a localized display name, BFF returns the localized display name.

If no localized value exists, BFF falls back to default display name.

### 34.10 Client Acceptance

Given a BFF imports the generated vNext client, it can perform typed operations without using generic `any` clients for first-party resources.

### 34.11 Watch Acceptance

Given a controller lists resources and receives a resourceVersion, it can watch or poll changes from that version and reconcile resources idempotently.

### 34.12 Controller Acceptance

Given a feature grant or quota changes, the entitlement snapshot controller updates effective entitlement state.

### 34.13 Audit Acceptance

Given a sensitive write occurs, vNext records:

* Actor
* Service account if applicable
* Tenant
* Scope if applicable
* Resource
* Action
* Decision
* Request ID
* Timestamp

---

## 35. Migration from Current Kernel

vNext should not directly copy the current kernel module model.

Useful patterns to preserve:

```text
module manifests
module dependency graph
migration tooling/ergonomics    (NOT central orchestration — see amendment)
schema ownership
tenant isolation ideas
event bus abstraction
transactional outbox
CLI patterns
health/telemetry/bootstrap conventions
```

> **v0.4 amendment (ADR-0007): migrations are `vnext-service`-owned.** Central
> "migration orchestration" is **not** preserved — it reintroduces a deploy-time
> coupling between the control plane and every domain service. Instead:
> - The control plane migrates only `control_plane.*` (its own primitives).
> - Each `vnext-service` owns and self-runs migrations for its own schema(s) on
>   deploy (init container / k8s `Job`), with a **schema-scoped DB role** (DDL
>   only on its own schema).
> - **No cross-schema hard FKs** (§14.4 hardens to a rule): independent, unordered
>   migrations mean no service may assume another's table exists.
>
> What *is* preserved is the migration *tooling/ergonomics*, packaged into the
> `vnext-service` SDK so each service gets first-class migration DX **without** the
> control plane running it.

Patterns to avoid preserving as final architecture:

```text
all modules compiled into one product binary
BFF importing domain internals
domain services communicating only through in-process registries
generic any-client as primary SDK
module lifecycle tied to application rebuild
```

Mapping:

```text
kernel module manifest
  -> vNext APIService / ResourceDefinition / ModulePackage manifest

kernel module enablement
  -> vNext FeatureGrant / Entitlement

kernel migrations
  -> domain-owned migrations

kernel hooks
  -> admission hooks / domain events / controllers

kernel internal readers
  -> generated clients and service APIs
```

---

## 36. Rollout Plan

### Phase 1: vNext Core

Build:

* Tenant model
* Principal model
* Membership model
* RBAC
* Feature grants
* Quotas
* Public IDs
* Labels and annotations
* Localized display names
* Audit logs
* Service accounts
* Signed delegation context
* Postgres control_plane schema
* Generated control-plane client

### Phase 2: API-Server Semantics

Build:

* Resource versions
* List pagination
* Resource events
* Basic watch/polling
* Status subresources
* Conditions
* Events
* Leases
* Controller manager basics

### Phase 3: Entitlement Runtime

Build:

* Effective entitlement snapshot
* Entitlement admission checks
* Quota admission checks
* Entitlement controller
* Subscription module integration contract

### Phase 4: Resource API Machinery

Build:

* ResourceDefinition registry
* APIService registry
* First-party proto contracts
* OpenAPI generation
* Typed client generation
* Dynamic client
* Generic resource table for runtime resources

### Phase 5: First Product Integration

Use HR OS Attendance to prove:

* BFF integration
* Signed request context
* Domain API authorization
* Feature entitlement enforcement
* Audit trail
* Public support IDs
* Labels/selectors
* Localized display names
* APIService registration

### Phase 6: Additional Products

Apply vNext to:

* Healthcare OS
* Telry
* Maritime SaaS
* AI Agents / Call Center Platform
* Insurance Core

---

## 37. Resolved Decisions and Remaining Open Decisions

### 37.1 Resolved Decisions

> **Each decision below is now governed by an ADR; see §0 and [`../adr/`](../adr/).
> Items 4, 5, and 9 were amended/superseded in v0.4.**

1. Use one `Tenant` resource with hierarchy stored in `spec.kind`, instead of separate `Tenant` and `Scope` resource kinds.
2. Use `metadata.publicID` for immutable opaque public IDs. `metadata.name` is the real resource name, and `metadata.uid` is the immutable internal system UID.
3. Define first-party APIs using protobuf plus Buf. Generated JSON Schema is the primary REST/admission validation contract.
4. *(Amended v0.4 — ADR-0006.)* The **implementation** of `ResourceDefinition`, dynamic clients, generic resource storage, and runtime/custom resources is phased after the core slice — but the discovery and client **contract** assumes dynamic types from day one. Dynamic types are a day-one architectural invariant, not a deferred afterthought.
5. *(Superseded v0.4 — ADR-0006.)* vNext **does aggregate**. It is the single API surface and discovery surface; consumers are backend-agnostic and never call domain servers directly. Aggregated types are proxied to their owning `vnext-service`. (The original "BFFs call domain APIs directly" is reversed.)
6. Domain APIs (`vnext-service`s) must strictly follow the vNext platform API contract on `/apis/*`.
7. v1 watch support is `list + resourceVersion + durable resource_events polling`. Streaming watch and informer/cache semantics are later targets.
8. Initial resource scoping is global for `Tenant`, `Principal`, `Role`, and `Permission`; organization or scope/branch scoped for `RoleBinding`, `FeatureGrant`, and `Quota`.
9. *(Superseded v0.4 — ADR-0004.)* Delegation no longer uses a signed delegation envelope. vNext authenticates every request (baked-in authn chain); across the aggregation boundary it uses **front-proxy identity injection** plus **TokenReview/SubjectAccessReview**.

Decisions added in v0.4 (each governed by an ADR; see §0):

10. **Build vNext fresh**, kernel as parts donor, strangler migration (ADR-0005).
11. **`Project` is killed**; hierarchy is `platform → organization → scope` (ADR-0007).
12. **Authentication is baked into the API server**: `iss`-keyed central + per-tenant `AuthProvider`, validate `iss`+`aud`, unique issuer (ADR-0004, supersedes ADR-0002).
13. **Aggregating single API surface + unified discovery**; strict-proxy reads with governed caching (ADR-0006).
14. **Deploy strictly on Kubernetes**; control-plane *data* in Postgres, *operational* plane k8s-native; Postgres lease table retired (ADR-0007).
15. **Migrations are `vnext-service`-owned**; control plane migrates only `control_plane.*`; schema-scoped DB roles; no cross-schema FKs (ADR-0007).
16. **RBAC inheritance via query-time ancestor check** (ADR-0001); **marketplace parked**, controller seam kept open (ADR-0003).

### 37.2 Remaining Open Decisions

1. What exact public ID prefix catalog should be used across all first-party resources?
2. Should audit logs remain only in Postgres initially or also stream to object storage/event bus?
3. Should `vnextctl` be included in MVP?
4. Should entitlement snapshots be updated synchronously on subscription changes or reconciled asynchronously?
5. Should labels be implemented on every resource from day one or only on core resources first?
6. Should localized display names be a field on all first-party resources or introduced resource by resource?
7. Should each product have generated product-specific client bundles or consume separate vNext/domain clients?
8. What are the exact scopes for `Membership`, `ServiceAccount`, `EntitlementSnapshot`, `APIService`, `AuditEvent`, and `Event`? (`Lease` removed — k8s-native, ADR-0007.)

> **v0.4 note:** items 1 and 8 **block the first implementation slice** — `pkg/publicid` cannot be rebuilt without the prefix catalog (1), and the schema + RBAC checks cannot be written strictly without the remaining scopes (8). Resolve these two first (each likely a short ADR). Items 2–7 do not block the core slice and can remain open.

---

## 38. Recommended First Implementation Slice

Build the smallest useful vNext slice:

```text
vnext-apiserver
  core tenants
  IAM principals/memberships/roles/rolebindings
  service accounts
  entitlements feature grants/quotas
  audit logs
  labels/annotations
  localized display names
  public IDs
  generated Go client
  signed request context
  Postgres resource versions/events
```

Use HR OS Attendance as the proof.

Required first flow:

```text
1. HR BFF verifies user auth.
2. HR BFF resolves vNext principal and membership.
3. HR BFF resolves organization and branch/scope.
4. HR BFF gets effective roles, permissions, and enabled features.
5. HR BFF calls Attendance API with signed context.
6. Attendance API verifies context and checks vNext authorization/entitlements.
7. Attendance API writes attendance domain state.
8. vNext audit records the action.
```

This proves vNext without overbuilding the entire platform.

---

## 39. Final Principle

vNext owns platform primitives.

Products own product experience.

Domain API servers own business behavior.

Controllers own reconciliation and derived state.

Postgres stores control-plane state.

Labels select and group resources.

Annotations explain and annotate resources.

Typed fields/spec/status hold authoritative state.

Public IDs help support and debugging.

Authorization is always enforced server-side.

Dynamic clients are used only for runtime/custom resources.

Typed clients are used for known first-party resources.

vNext should be depended on as an API server, not imported as a framework and not bypassed as a database.
