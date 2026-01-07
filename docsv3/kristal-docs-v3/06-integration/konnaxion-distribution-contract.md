# Konnaxion distribution contract (v3 integration)

## Status

Draft (v3 integration contract)

## Purpose

This document defines the contract for how **Konnaxion** distributes, verifies, caches, and activates **Kristal v3 Runtime Packs** (and associated metadata) for offline/low-bandwidth operation.

Konnaxion’s responsibilities in the ecosystem:

* distribute **versioned offline packages**
* enforce **integrity verification** (signatures/hashes) where declared
* provide **predictable activation/rollback/downgrade behavior**
* surface pack provenance and version metadata to higher-level modules (search, navigation, knowledge delivery, curation)

Normative keywords: MUST, MUST NOT, SHOULD, SHOULD NOT, MAY.

---

## 1. Scope and non-goals

### 1.1 In scope

* Pack packaging format and required metadata
* Trust roots and verification rules (fail-closed)
* Versioning, activation, rollback, downgrade prevention
* Offline caching behavior and storage layout expectations
* Distribution channels (tenant/environment scoped)
* Telemetry and feedback signals (as operational signals, not fact mutation)

### 1.2 Out of scope

* How packs are built (Orgo / compiler)
* How facts are debated/curated (Ekoh/Konsensus/Smart Vote policies)
* Query semantics inside the Runtime Pack (covered by query contract)
* Full device management / MDM integration (deployment-specific)

---

## 2. Artifact types and identifiers

### 2.1 Runtime Pack bundle (mandatory)

A distributable unit MUST include:

* `pack_manifest.json` (mandatory)
* pack payload files (Parquet tables, indexes, filters, dictionaries, etc.)
* optional `signatures` for the manifest and/or bundle
* optional `pack_index.json` (channel-level index; see Section 3)

### 2.2 Required IDs (mandatory)

Each pack bundle MUST be uniquely identified by:

* `pack_id` (content-addressed, preferred) OR a deterministic `pack_content_id`
* `source_kristal_id` (the Exchange kristal_id used to compile the pack)
* `build_id` (compiler build id)
* `pack_version` (see Section 4)

Konnaxion MUST treat `pack_id` as the primary identity if present.

---

## 3. Distribution channel and indexes

### 3.1 Channels (mandatory)

Packs MUST be distributed via **channels** that are scoped at minimum by:

* `tenant_id`
* `environment` (e.g., `prod`, `staging`)
* optional `region` or `audience_segment`

Konnaxion MUST NOT mix trust roots or activation rules across channels unless explicitly configured.

### 3.2 Channel pack index (recommended)

Konnaxion SHOULD consume a signed channel index (example: `pack_index.json`) containing:

* `channel_id`
* `latest` pointer(s)
* `pinned` pointer(s)
* optional `minimum_allowed_version`
* optional `revoked` list
* metadata for delta updates (optional profile)

If present and signed, the index MUST be verified using the channel’s trust roots (fail-closed).

---

## 4. Versioning and activation semantics

### 4.1 Version fields (mandatory)

Konnaxion MUST treat pack versions as structured, not ad hoc strings. Minimum required fields:

* `pack_version` (monotonic within a channel)
* `source_kristal_id` (content lineage)
* `compiler_version`
* `created` timestamp

### 4.2 Activation rule (mandatory)

Konnaxion MUST only activate a pack when:

1. pack bundle integrity is verified (if declared)
2. manifest parses and passes schema validation
3. required files referenced by manifest exist
4. pack is compatible with the client runtime (contract versions match)

Activation MUST be an atomic switch from old pack to new pack (no partial activation).

### 4.3 Downgrade prevention (mandatory)

Konnaxion MUST implement downgrade prevention in at least one deterministic form:

* **Monotonic version rule** (recommended): do not activate a pack with `pack_version` lower than the currently active version for that channel, unless explicitly permitted by policy (e.g., emergency rollback).
* **Revocation-aware rule**: do not activate packs listed as revoked in a verified channel index.

If rollback is permitted, it MUST occur through an explicit rollback action (Section 5), not silently.

---

## 5. Rollback behavior (mandatory)

### 5.1 Rollback modes

Konnaxion MUST support at least one rollback mode:

* **Pinned rollback**: activate a previously pinned known-good pack
* **Last-known-good rollback**: activate the most recent previously active pack that is still present and verified

### 5.2 Rollback triggers (deployment policy)

Rollback MAY be triggered by:

* explicit operator action (recommended)
* verified index updates (e.g., “latest revoked”)
* local runtime health signals (optional profile; e.g., repeated query failures)

Rollback MUST be deterministic given the same trigger event sequence.

---

## 6. Integrity verification (fail-closed)

### 6.1 Verification requirements

If the pack manifest or bundle declares any of:

* signatures
* hashes
* signer identity (`kid`)

then Konnaxion MUST:

* verify them before activation
* fail closed on any verification failure

### 6.2 Trust roots (mandatory)

Konnaxion MUST pin trust roots per channel (tenant/environment), and MUST be able to verify offline.

Trust roots MUST NOT be fetched over the network at activation time as a dependency for correctness; they must be pre-provisioned or cached securely.

### 6.3 Verification scope (recommended)

Recommended verification order:

1. verify channel index (if used)
2. verify pack manifest signature(s)
3. verify bundle file hashes referenced by manifest (optional profile; can be expensive)
4. only then activate

---

## 7. Offline caching and storage contract

### 7.1 Storage layout (recommended)

Konnaxion SHOULD store packs in a layout that supports:

* multiple installed versions per channel
* atomic activation
* garbage collection with pinning rules

Example layout:

* `/<tenant>/<env>/<channel>/packs/<pack_id>/...`
* `/<tenant>/<env>/<channel>/active -> <pack_id>`

### 7.2 Cache policy (mandatory to define)

Konnaxion MUST define deterministic cache policies:

* max disk usage per channel
* eviction policy (e.g., LRU with pinned exceptions)
* minimum set of pinned packs to retain (e.g., active + last-known-good)

### 7.3 Low-bandwidth / offline behavior (mandatory)

If offline, Konnaxion MUST:

* continue serving from the active pack
* not attempt activation requiring network-dependent trust roots
* surface “stale pack” metadata (optional UX requirement)

---

## 8. Compatibility checks (mandatory)

Before activation, Konnaxion MUST verify:

* `query_contract_version` supported by the client runtime
* policy selections are supported (or the pack is rejected deterministically)
* required projections/data are present for the consuming module(s) (e.g., if a module requires truthy projection)

If incompatible:

* do not activate
* emit a deterministic error code and diagnostic payload

---

## 9. Telemetry and feedback signals (non-mutating)

Konnaxion MAY emit operational signals to Orgo such as:

* download/verification failures
* activation success/failure
* query runtime errors and performance summaries
* pack usage metrics (counts, not raw facts)

These signals MUST NOT mutate Kristal Exchange directly; they should create Cases/Tasks or distribution adjustments.

---

## 10. Conformance tests (required)

A Konnaxion implementation claiming conformance MUST provide tests for:

* signature verification success/failure (fail-closed)
* atomic activation and no partial state
* downgrade prevention behavior
* rollback behavior and determinism
* offline behavior (serve active pack; no network dependency for correctness)
* cache eviction respects pinned packs

---

## 11. Open questions

* Do we require bundle-level file hashing in core, or keep it as a profile due to cost?
* Do we standardize delta update manifests for packs (likely a profile)?
* How do we represent revocation lists (index vs separate signed document)?
