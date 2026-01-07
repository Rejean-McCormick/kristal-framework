# Multi-tenancy boundaries (global content IDs + tenant access control layering)

## Status
Draft (Kristal v3)

## Purpose
Define the **multi-tenancy boundary model** for Kristal v3 so that:
- **IDs remain globally content-addressed** (same content → same `kristal_id`)
- **tenant isolation** is enforced by **access control + signing keys + distribution channels**
- operational systems (Orgo/Konnaxion) can safely manage builds, governance, and offline distribution without cross-tenant leakage

This document is normative where it uses MUST/SHOULD.

## Definitions
- **Tenant**: an isolated administrative/security domain (org, deployment, customer, jurisdiction).
- **Exchange**: canonical Kristal artifact; content-addressed source of truth.
- **Runtime Pack**: derived offline-executable artifact; distributed to clients/nodes.
- **Global content ID**: ID derived only from canonicalized content (not from tenant metadata).
- **Access layer**: authorization + distribution controls that determine who can obtain or use an artifact.
- **Signing domain**: key material and trust roots used to sign artifacts in a tenant/environment.

## Core model (v3)

### 1) Global content IDs (MUST)
- `kristal_id` MUST be computed solely from **canonicalized Exchange content** (JCS), excluding signatures and other non-content fields.
- The same Exchange content produced in two tenants MUST yield the same `kristal_id`.

Rationale: enables caching, deduplication, reproducible builds, and cross-implementation comparability.

### 2) Tenant isolation is layered above IDs (MUST)
Tenant separation MUST be enforced by:
- **access control** (who can fetch/see/use),
- **signing keys + trust roots** (who can trust/verify),
- **distribution channels** (where/how packs are delivered),
not by changing the content hash or `kristal_id`.

### 3) Tenant metadata MUST NOT influence core hashes (MUST)
Tenant identifiers, ACLs, approvals, workflow state, and distribution status MUST NOT be included in:
- Exchange content canonicalization for `kristal_id`,
- Exchange `content_hash`,
- Runtime Pack content hash (unless a separate declared integrity profile explicitly includes them—which is discouraged).

Tenant metadata belongs in Orgo/Konnaxion control-plane records, not in Kristal payload identity.

## Artifact visibility and authorization

### 4) Access decisions (MUST)
Implementations MUST enforce access control at these boundaries:
- **Exchange fetch / read**
- **Runtime Pack fetch / read**
- **Runtime Pack activation / use on client/node** (especially for offline caches)

Access checks MUST be tenant-scoped and MUST prevent:
- cross-tenant enumeration of artifact existence,
- inference via error messages (use consistent “not found or not authorized” responses where appropriate).

### 5) Content-addressability and privacy (SHOULD)
Because global IDs can enable cross-domain correlation (two tenants independently producing identical content share the same `kristal_id`), deployments SHOULD consider:
- restricting exposure of raw `kristal_id` values outside authorized contexts,
- using opaque handles in UI/API responses where correlation risk matters,
- ensuring error handling does not reveal whether a given `kristal_id` exists in another tenant.

If a deployment explicitly requires “no cross-tenant correlation,” it MUST use an additional tenant-scoped indirection layer (see Section 9) while keeping `kristal_id` global internally.

## Signing, trust roots, and tenant domains

### 6) Tenant-scoped signing domains (MUST)
- Each tenant/environment MUST have a defined **trust root set** (public keys / cert roots) used to verify signatures.
- Artifacts MAY be signed by tenant-specific keys even if IDs are global.
- Verification MUST use the tenant’s pinned trust roots.

### 7) Same content, different signatures (MUST)
It MUST be possible for:
- Tenant A and Tenant B to publish the same Exchange (same `kristal_id`)
- but sign it with different keys.

Signatures MUST be treated as **metadata** over the declared `content_hash` and MUST NOT change the `kristal_id`.

### 8) Distribution channel isolation (MUST)
Runtime Pack distribution MUST be isolated per tenant via:
- separate package indexes / feeds, or
- tenant-authenticated endpoints, or
- tenant-specific offline bundle channels.

Clients MUST NOT accept packs from untrusted channels even if signatures verify under some other tenant’s trust roots.

## Control-plane vs data-plane division (Orgo / Konnaxion)

### 9) Orgo is the control plane (MUST)
Orgo SHOULD store (tenant-scoped):
- workflow state (cases/tasks/approvals),
- who requested or approved builds,
- distribution status,
- audit logs,
- policy configuration per tenant,
- key references (KIDs) and trust-root sets.

Kristal artifacts SHOULD store only:
- content, manifests, reproducibility policy selections, and signatures.

### 10) Konnaxion distribution and offline caching (MUST)
Konnaxion clients/nodes MUST:
- verify signatures against tenant trust roots before using a pack,
- maintain tenant-separated caches (no cross-tenant cache reuse unless explicitly configured),
- enforce rollback/downgrade prevention within a tenant channel.

## Reproducibility and comparability across tenants

### 11) Reproducibility remains tenant-independent (MUST)
Given identical inputs and policy selections, two tenants compiling the same Exchange content MUST be able to produce:
- identical Exchange content hashes and `kristal_id`,
- byte-identical exports (if same export profile and serialization rules),
- reproducible Runtime Packs when the same portable policies are used.

### 12) Policy selection is tenant-configurable but recorded (MUST)
Tenants MAY choose different portable policies (ordering, row groups, filter parameters) for Runtime Packs.
When they do, those policy selections MUST be recorded in the manifests so packs remain comparable and reproducible within their policy class.

## Threat considerations (non-exhaustive)

### Correlation risk (SHOULD mitigate)
Global IDs can reveal that two tenants have identical content if IDs are exposed publicly.
Mitigations:
- opaque handles externally,
- tenant-authenticated lookup,
- do not expose IDs in URLs.

### Downgrade/rollback attacks (MUST mitigate)
- Clients MUST apply a per-tenant rollback/downgrade prevention policy (see `07-security/downgrade-rollback-policy.md`).
- Signed “latest” pointers or pinned release channels SHOULD be used.

### Cross-tenant cache pollution (MUST prevent)
- Client caches MUST be partitioned by tenant.
- Package indices MUST be tenant-authenticated.

## Recommended implementation patterns (non-normative)
- Use a **tenant-scoped artifact registry** that maps:
  - `tenant_artifact_handle` → (`kristal_id`, allowed channels, trust roots, policy class)
- Keep `kristal_id` as an internal canonical key, but avoid exposing it directly in unauthenticated contexts.
- Use structured logs with `{tenant_id, build_id, kristal_id}` but ensure log access is tenant-scoped.

## Open questions (to finalize)
- Do we require opaque handles everywhere externally, or only for public/unauthenticated contexts?
- Do we standardize a tenant registry schema as non-normative guidance (recommended) or a formal profile (likely non-normative)?
- How do we handle “shared public Kristals” intended to be cross-tenant by design (e.g., public education packs)? (Likely: separate public tenant/channel with its own trust roots.)
