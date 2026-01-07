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

## 1. Terminology

* **Exchange**: the canonical Kristal Exchange JSON artifact.
* **Runtime Pack**: the derived offline-executable artifact (not defined here).
* **canonical_json**: the canonical byte representation of an object after applying this spec’s canonicalization rules.
* **Hash target**: the exact JSON object that is canonicalized and hashed to produce an ID.
* **JCS**: JSON Canonicalization Scheme, RFC 8785, used as the normative canonicalization method in v3. 

Normative keywords: MUST, MUST NOT, SHOULD, SHOULD NOT, MAY.

---

## 2. Normative canonical_json (v3)

### 2.1 Canonicalization method (mandatory)

`canonical_json` MUST be produced using **RFC 8785 (JCS)**. 

### 2.2 Canonicalization profile recording (mandatory)

Every Exchange artifact and every Runtime Pack manifest MUST record:

* `canonicalization_profile` (string identifier)
* `canonicalization_version` (string or integer)

This is required so hashes remain comparable across toolchains. 

**Default profile identifier (recommended):**

* `canonicalization_profile = "kristal.v3:jcs-rfc8785"`
* `canonicalization_version = "1"`

Implementations MAY introduce additional profiles, but MUST make them explicit and MUST NOT claim v3 core conformance under an unspecified profile.

---

## 3. Hash target selection

### 3.1 Hash exclusions (mandatory)

To compute IDs, implementations MUST derive a **hash target object** from the Exchange artifact by removing:

1. `kristal_id` (the output field)
2. any declared signatures / attestations fields (wherever they appear)

This aligns with v0.1/v2 guidance to hash “exchange_without_signatures” while keeping signatures as optional overlays.   

**Note:** This doc defines *what is removed* to avoid circular hashing and to make signing/verifying well-defined.

### 3.2 Optional “stable content vs build metadata” guidance (non-normative)

v3 encourages reproducibility and comparable IDs across independent builds.  If you include volatile metadata (wall-clock timestamps, transient metrics) in the hashed region, the `kristal_id` will change even when knowledge content does not.

Recommendation:

* keep volatile compilation telemetry in a clearly separated section (e.g., `manifest.build`) and (if you choose) exclude it **only via a declared profile** (do not do silent exclusions).

---

## 4. kristal_id computation (mandatory)

### 4.1 Definition

Kristal Exchange uses content-addressing:

`kristal_id = sha256(canonical_json(exchange_without_signatures))`  

In v3, the phrase “exchange_without_signatures” is formalized as the **hash target** in Section 3.

### 4.2 Algorithm (normative)

Given an Exchange JSON object `E`:

1. Produce `T = hash_target(E)` by:

   * removing `kristal_id`
   * removing signatures/attestations fields (per Section 3.1)
2. Serialize `T` to bytes `B = JCS(T)` using RFC 8785. 
3. Compute `H = SHA-256(B)`.
4. Encode `H` as **lowercase hex**.
5. Set:

   * `kristal_id = "sha256:" + hex(H)`

### 4.3 Acceptance criteria (mandatory)

* Two independent implementations MUST produce identical `kristal_id` for the same Exchange artifact under the same `canonicalization_profile` and `canonicalization_version`. 

---

## 5. statement_id computation (optional but recommended)

Kristal v0.1 already supports optional statement identifiers:
`statement_id = hash(subject + property + value + qualifiers + reference_ids)` 

### 5.1 Purpose

A stable `statement_id` enables: deduplication, merge operations, and fine-grained provenance tracking. 

### 5.2 Normative definition (if enabled)

If `statement_id` is present, the producer MUST compute it deterministically from a **statement hash target** that includes:

* `subject` (QID)
* `property` (PID)
* `value` (fully normalized literal or QID)
* `qualifiers` (including all qualifier snaks)
* `reference pointers` (e.g., `evidence_id` or normalized reference snaks)

Procedure (recommended):

1. Build statement hash target object `S`.
2. Canonicalize with the same `canonicalization_profile` as the Exchange. 
3. Hash with SHA-256.
4. Encode as `sha256:<hex>`.

### 5.3 Ordering rules (normative when relevant)

Where statement substructures are semantically sets (qualifiers, references), implementations MUST sort them deterministically before hashing the statement to avoid ID drift. This aligns with earlier v0.1 canonicalization intent (sorting where order is not semantically meaningful). 

---

## 6. Optional RDF-level hashing: rdf_hash (profile)

v3 adds an optional RDF integrity mode for exports:

* `rdf_hash` computed using **RDF Dataset Canonicalization (RDFC-1.0)** from canonical N-Quads. 

This is **optional** and enabled only under an explicit profile (see `05-profiles/profile-rdf-integrity-rdfc.md` in the doc plan). When enabled, CI gating against the RDFC test suite is expected, with resource limits. 

---

## 7. Optional human-verifiable ID representations (profile)

v3 allows an optional Trusty/ni-URI-style representation as a human-usable, verifiable form of `kristal_id` (or `rdf_hash` in RDF mode). 

Nanopublication guidance similarly treats Trusty-URI-like commitments as integrity keys. 

---

## 8. Signing workflow dependency (normative reference)

The signing/verification workflow is:

**remove signatures → canonicalize → hash/verify → sign** 

Fail-closed semantics for declared hashes/signatures are specified in `01-core-spec/signatures-trust.md`. (This file defines the ID and canonicalization prerequisites that signing depends on.)
