# 01-core-spec/ids-canonicalization-hashing.md

## Status

Draft (v3)

## Purpose

This document specifies the **normative** rules for:

* `canonical_json` (how Exchange objects are canonicalized)
* `kristal_id` (how content-addressed IDs are computed)
* optional `statement_id` (how statement-level IDs are computed)
* optional `rdf_hash` (RDF-level hash for deterministic exports)

v3’s goal is that two independent implementations produce identical IDs for identical artifacts, using a portable canonicalization and hashing pipeline.

---

## 0. Scope note: Exchange payload vs manifests (normative)

This document defines ID computation over the **Kristal Exchange payload** (the canonical knowledge artifact).

**Manifests** (e.g., `02-schemas/exchange-manifest.schema.json`, `02-schemas/runtime-pack-manifest.schema.json`) are **sidecars** that record build inputs/parameters, timestamps, and declared integrity material. Manifests are **not part of the Exchange hash target** unless a profile explicitly says otherwise.

**Implication (core conformance):**
* Wall-clock timestamps and other volatile compilation telemetry MUST NOT appear inside the Exchange payload hashed region; they belong in sidecar manifests / operational logs.

---

## 1. Terminology

* **Exchange (payload)**: the canonical Kristal Exchange JSON knowledge artifact that is content-addressed.
* **Exchange Manifest**: the sidecar manifest that records reproducibility and integrity declarations for an Exchange.
* **Runtime Pack Manifest**: the sidecar manifest for a compiled offline Runtime Pack.
* **canonical_json**: the canonical byte representation of an object after applying this spec’s canonicalization rules.
* **Hash target**: the exact JSON object that is canonicalized and hashed to produce an ID.
* **JCS**: JSON Canonicalization Scheme, RFC 8785, used as the normative canonicalization method in v3.

Normative keywords: MUST, MUST NOT, SHOULD, SHOULD NOT, MAY.

---

## 2. Normative canonical_json (v3)

### 2.1 Canonicalization method (mandatory)

`canonical_json` MUST be produced using **RFC 8785 (JCS)**.

### 2.2 Canonicalization profile recording (mandatory)

Every Exchange payload and every Runtime Pack manifest MUST record:

* `canonicalization_profile` (string identifier)
* `canonicalization_version` (string)

This is required so hashes remain comparable across toolchains.

**Default profile identifier (recommended for v3 core):**

* `canonicalization_profile = "kristal.v3:jcs-rfc8785"`
* `canonicalization_version = "1"`

Implementations MAY introduce additional profiles, but MUST make them explicit and MUST NOT claim v3 core conformance under an unspecified profile.

---

## 3. Hash target selection

### 3.1 Hash exclusions (mandatory)

To compute IDs, implementations MUST derive a **hash target object** from the Exchange payload by removing:

1. `kristal_id` (the output field)
2. `content_hash` (if present in the payload as a stored digest/output)
3. any declared signatures / attestations fields

**Canonical placement rule (core):**
* Signatures MUST be carried in a top-level `signatures` array on the payload when present.
* For forward/defensive compatibility, when forming the hash target, implementations MUST remove any fields whose key is exactly `signatures` or `attestations` anywhere they appear.

This aligns with the pipeline rule: **remove signatures → canonicalize → hash/verify → sign**.

### 3.2 No implicit exclusions (mandatory)

Except for the exclusions in §3.1, implementations MUST NOT silently exclude additional fields from the hash target.

If an implementation needs a “stable content vs build metadata” split that changes the hashed region, it MUST be done only via an explicit, recorded profile (i.e., a different `canonicalization_profile` / `canonicalization_version`, or a separate named hashing profile if defined by a v3 profile document).

---

## 4. kristal_id computation (mandatory)

### 4.1 Definition

Kristal Exchange uses content-addressing:

`kristal_id = "sha256:" + hex(SHA-256(JCS(hash_target(E))))`

Where `hash_target(E)` is defined in Section 3.

### 4.2 Algorithm (normative)

Given an Exchange JSON object `E`:

1. Produce `T = hash_target(E)` by applying §3.1.
2. Serialize `T` to bytes `B = JCS(T)` using RFC 8785.
3. Compute `H = SHA-256(B)`.
4. Encode `H` as **lowercase hex**.
5. Set:

   * `kristal_id = "sha256:" + hex(H)`

### 4.3 Acceptance criteria (mandatory)

* Two independent implementations MUST produce identical `kristal_id` for the same Exchange payload under the same `canonicalization_profile` and `canonicalization_version`.

---

## 5. statement_id computation (optional but recommended)

### 5.1 Purpose

A stable `statement_id` enables: deduplication, merge operations, and fine-grained provenance tracking.

### 5.2 Normative definition (if enabled)

If `statement_id` is present, the producer MUST compute it deterministically from a **statement hash target** that includes:

* `subject` (QID)
* `property` (PID)
* `value` (fully normalized literal or QID)
* `qualifiers` (all qualifier snaks)
* `references` (reference pointers or normalized reference snaks, if present)

Procedure (normative):

1. Build statement hash target object `S`.
2. Canonicalize with the same `canonicalization_profile` and `canonicalization_version` as the Exchange payload.
3. Hash with SHA-256.
4. Encode as `sha256:<hex>`.

### 5.3 Ordering rules (normative when relevant)

Where statement substructures are semantically sets (qualifiers, references), implementations MUST sort deterministically before hashing:

* Primary sort key: predicate ID (PID) ascending (lexicographic)
* Secondary: canonicalized value representation (lexicographic over canonical_json bytes)
* Tertiary: stable tie-breaker (e.g., canonical_json of the whole snak)

---

## 6. Optional RDF-level hashing: rdf_hash (profile)

v3 adds an optional RDF integrity mode for exports:

* `rdf_hash` computed using **RDF Dataset Canonicalization (RDFC-1.0)** from canonical N-Quads.

This is optional and enabled only under an explicit profile (see `05-profiles/profile-rdf-integrity-rdfc.md`). When enabled, CI gating against an RDFC test suite is expected, with resource limits.

---

## 7. Optional human-verifiable ID representations (profile)

v3 allows an optional Trusty/ni-URI-style representation as a human-usable, verifiable form of `kristal_id` (or `rdf_hash` in RDF mode).

---

## 8. Signing workflow dependency (normative reference)

The signing/verification workflow is:

**remove signatures → canonicalize → hash/verify → sign**

Fail-closed semantics for declared hashes/signatures are specified in `01-core-spec/signatures-trust.md`. This file defines the ID and canonicalization prerequisites that signing depends on.