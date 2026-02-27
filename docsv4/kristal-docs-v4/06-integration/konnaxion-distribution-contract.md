# Konnaxion distribution contract (v3 integration)

## Status

Draft (v3 integration contract)

## Purpose

This document defines the contract for how **Konnaxion** distributes, verifies, caches, and activates **Kristal v4 Runtime Packs** (and associated metadata) for offline/low-bandwidth operation.

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

* `runtime-pack-manifest.json` (mandatory; MUST conform to `02-schemas/runtime-pack-manifest.schema.json`)
  * Backward compatibility: consumers MAY accept `pack_manifest.json` as an alias filename, but emitters SHOULD publish `runtime-pack-manifest.json`.
* pack payload files (Parquet tables, indexes, filters, dictionaries, etc.)
* optional `signatures` envelope(s) for the manifest and/or bundle
* optional signed channel index (example: `pack_index.json`; see Section 3)

### 2.2 Required identifiers (mandatory)

Each Runtime Pack bundle intended for distribution MUST be uniquely and unambiguously identified by the following metadata (carried either in the Runtime Pack Manifest and/or in the signed channel index):

* `artifact_type = "runtime_pack"`
* `artifact_id` (content-addressed)
  * Canonical: `runtime_pack_id`
  * Compatibility: other surfaces MAY call this `pack_id` (treated as a synonym)
* `release_id` (monotonic within a channel)
  * Compatibility: older surfaces MAY call this `pack_version`
* `build_id`
* `exchange_ref.exchange_id` (the Exchange ID used to compile the pack)
  * Compatibility: older surfaces MAY call this `source_kristal_id`
* `created_at` (for audit/debugging; MUST NOT be used alone for monotonicity)
* `channel` (channel identifier; see §3.1)
* `signer_key_id` (the signing key identifier)
  * Compatibility: older surfaces MAY call this `kid`

Konnaxion MUST treat `runtime_pack_id` as the primary identity for pack selection, verification binding, caching, and activation.

---

## 3. Distribution channel and indexes

### 3.1 Channels (mandatory)

Packs MUST be distributed via **channels** that are scoped at minimum by:

* `tenant_id`
* `environment` (e.g., `prod`, `staging`)
* optional `region` or `audience_segment`

Konnaxion MUST NOT mix trust roots, verification policy, or downgrade/rollback state across channels unless explicitly configured.

### 3.2 Signed channel index (“what to install”) (recommended)

Konnaxion SHOULD consume a signed distribution index per channel (example: `pack_index.json`). If present, it MUST be verified using the channel’s pinned trust roots (fail-closed).

A channel index MUST include at minimum:

* `channel_id`
* `current` pointer:
  * `release_id`
  * `artifact_id` (i.e., `runtime_pack_id`)
* signer identity + signature envelope

A channel index SHOULD additionally include:

* optional staged/canary pointers (cohort targeting is implementation-defined)
* `pinned` pointer(s) (known-good packs)
* `minimum_allowed_release_id`
  * Compatibility: MAY accept `minimum_allowed_version` as an alias
* `revoked` list (artifact ids and/or release ids), and/or a revocation epoch marker
* optional rollback authorization record(s) (see §5)

If a verified channel index is present, Konnaxion MUST treat it as the primary control plane for “what to activate”, subject to local safety enforcement (downgrade prevention, compatibility checks).

---

## 4. Versioning and activation semantics

### 4.1 Release/version fields (mandatory)

Konnaxion MUST treat release/version identifiers as structured, not ad hoc strings.

* The authoritative monotonic field is `release_id` (monotonic within a channel).
* Implementations MUST choose a monotonic scheme per channel (recommended: integer sequence) and MUST define deterministic comparison rules.
* Compatibility: `pack_version` MAY be accepted as an alias for `release_id`, but Konnaxion MUST normalize to a single internal `release_id` for enforcement.

### 4.2 Activation rule (mandatory)

Konnaxion MUST only activate a pack when all of the following are true:

1. **Integrity verification passes** (as declared; fail-closed)
2. Runtime Pack Manifest:
   * parses and passes schema validation
   * records deterministic build intent where required (`build.deterministic = true` for core-conformant packs)
   * includes a complete file inventory with hashes sufficient for offline verification
3. Required files referenced by the manifest exist in the bundle
4. Pack is compatible with the client runtime (contract versions / required profiles match)

Activation MUST be an atomic switch from old pack to new pack (no partial activation).

### 4.3 Downgrade prevention (mandatory)

Konnaxion MUST implement downgrade prevention with deterministic, persisted state.

Minimum required rule:

* Konnaxion MUST NOT activate an artifact with a lower `release_id` than the **highest previously activated** release in the same channel, unless a **policy-authorized rollback** is present.
* Konnaxion MUST persist (per channel):
  * `highest_activated_release_id[channel]`
  * (recommended) `highest_seen_release_id[channel]`
  * last successful verification metadata (artifact_id, signer_key_id, timestamp)

Additional required rule (substitution safety):

* If a pack is received with the same `release_id` but a different `artifact_id`, Konnaxion MUST treat this as a hard error unless an explicit reissue authorization is present in a verified channel index.

Revocation-aware rule:

* Konnaxion MUST NOT activate packs listed as revoked in a verified channel index (or equivalent verified revocation mechanism).

---

## 5. Rollback behavior (mandatory)

### 5.1 Rollback modes

Konnaxion MUST support at least one rollback mode:

* **Pinned rollback**: activate a previously pinned known-good pack
* **Last-known-good rollback**: activate the most recent previously active pack that is still present and verified

### 5.2 Rollback authorization (mandatory where rollback is allowed)

Rollback MUST be explicit and MUST be policy-authorized. Authorization MAY be conveyed via:

* a verified channel index that includes an explicit rollback authorization record, and/or
* an explicit operator action that is itself auditable and policy-gated (deployment-specific)

If rollback is permitted, it MUST NOT occur silently.

### 5.3 Rollback triggers (deployment policy)

Rollback MAY be triggered by:

* explicit operator action (recommended)
* verified index updates (e.g., “current revoked”, rollback authorization added)
* local runtime health signals (optional profile; e.g., repeated query failures)

Rollback MUST be deterministic given the same trigger event sequence and the same verified inputs.

---

## 6. Integrity verification (fail-closed)

### 6.1 Verification requirements

If the Runtime Pack Manifest, channel index, or bundle declares any of:

* signatures (in a detached or top-level `signatures` envelope)
* hashes / file inventory hashes
* signer identity (`signer_key_id` / `kid`)

then Konnaxion MUST:

* verify them before activation (or, for large bundles, before first use of the referenced bytes)
* fail closed on any verification failure or ambiguity

### 6.2 Trust roots (mandatory)

Konnaxion MUST pin trust roots per channel (tenant/environment), and MUST be able to verify offline.

Trust roots MUST NOT be fetched over the network at activation time as a dependency for correctness; they must be pre-provisioned or cached securely.

### 6.3 Verification scope and ordering (recommended)

Recommended verification order:

1. verify channel index (if used)
2. verify Runtime Pack Manifest signature(s)
3. verify bundle file hashes as declared in the manifest’s file inventory
4. only then activate

---

## 7. Offline caching and storage contract

### 7.1 Storage layout (recommended)

Konnaxion SHOULD store packs in a layout that supports:

* multiple installed versions per channel
* atomic activation
* garbage collection with pinning rules
* persisted per-channel safety state (`highest_activated_release_id`, etc.)

Example layout:

* `/<tenant>/<env>/<channel>/packs/<runtime_pack_id>/...`
* `/<tenant>/<env>/<channel>/active -> <runtime_pack_id>`
* `/<tenant>/<env>/<channel>/state.json` (persisted safety metadata)

### 7.2 Cache policy (mandatory to define)

Konnaxion MUST define deterministic cache policies:

* max disk usage per channel
* eviction policy (e.g., LRU with pinned exceptions)
* minimum set of retained packs (e.g., active + last-known-good + pinned set)

### 7.3 Low-bandwidth / offline behavior (mandatory)

If offline, Konnaxion MUST:

* continue serving from the active pack
* not attempt activation requiring network-dependent trust roots
* surface “stale pack” metadata (optional UX requirement)

---

## 8. Compatibility checks (mandatory)

Before activation, Konnaxion MUST verify (from the Runtime Pack Manifest):

* query contract compatibility (contract id/version supported by the client runtime)
* required optional profiles are supported (or the pack is rejected deterministically)
* required data projections are present for the consuming module(s)
* policy selections are within the allowed portable policy set (or the pack is rejected deterministically)

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
* file inventory hash verification behavior (fail-closed)
* atomic activation and no partial state
* downgrade prevention behavior, including persisted `highest_activated_release_id`
* “same release_id, different artifact_id” substitution rejection
* rollback behavior, including authorization gating and determinism
* offline behavior (serve active pack; no network dependency for correctness)
* cache eviction respects pinned packs and retains last-known-good

---

## 11. Open questions

* Do we require pre-activation verification of all bundle file hashes in core, or allow “verify-on-first-use” as a documented profile?
* Do we standardize delta update manifests for packs (likely a profile)?
* How do we represent revocation lists (index vs separate signed document), and how should revocation epochs be handled offline?
* Do we require a single canonical JSON filename for the Runtime Pack Manifest (`runtime-pack-manifest.json`), or keep dual naming permanently?