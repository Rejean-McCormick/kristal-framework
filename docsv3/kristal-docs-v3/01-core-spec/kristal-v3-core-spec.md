# Kristal v3 Core Specification

## Status
Draft (normative core)

## Purpose
Define the **minimal normative requirements** for Kristal v3 conformance, with a strong focus on:
- cross-implementation interoperability,
- deterministic identity and integrity verification,
- reproducible compilation,
- clear inter-system contracts for the Kristal stack.

Everything not explicitly required here is either:
- specified as an **optional standardized profile** (see `05-profiles/`), or
- provided as **non-normative implementation guidance** (see `08-ops/`).

## Scope and non-goals
### In scope
- Canonicalization, hashing, signing, and verification semantics
- Minimal reproducibility requirements and manifest content
- Inter-system contracts: Claim-IR → Resolution → Validation → Exchange → Runtime Pack
- Deterministic baseline exports (JSON-LD / RDF export profiles) sufficient for interoperability

### Out of scope (non-goals)
- Full SPARQL semantics for Runtime Packs
- Operational patterns as schema-level objects (DLQ, circuit breaker, canary, etc.)
- Any specific runtime performance implementation beyond the **portable policy set** (enumerated policies are specified elsewhere)

## Normative language
The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are to be interpreted as normative requirements.

## Conformance overview
An implementation is **Kristal v3 Core Conformant** if it:
1. Implements the **canonicalization + identity** rules in this document.
2. Implements **fail-closed integrity verification** semantics.
3. Emits and honors a **minimal reproducibility manifest**.
4. Enforces the **inter-system interface contracts** (schemas + deterministic rules).
5. Produces **deterministic baseline exports** when enabled.

Optional profiles do not affect core conformance unless the implementation claims support for them. If an implementation claims a profile, it MUST satisfy that profile’s requirements.

---

# 1. Artifact model

Kristal v3 defines two primary artifact classes:

1. **Kristal Exchange Artifact (Exchange)**
   - Canonical, auditable, content-addressed source of truth.
   - Mergeable representation of validated knowledge claims.

2. **Kristal Runtime Pack (Runtime Pack)**
   - Derived, indexed, offline-executable representation.
   - Constrained query semantics; NOT full SPARQL.

Both artifact classes MUST have an associated **Manifest**.

---

# 2. Canonicalization and identity (mandatory)

## 2.1 Canonical JSON
- `canonical_json` MUST be defined as **RFC 8785 JSON Canonicalization Scheme (JCS)**.
- Implementations MUST NOT introduce additional canonicalization steps that change byte output beyond JCS.

## 2.2 Content-addressed IDs
### 2.2.1 kristal_id for Exchange
- `kristal_id` MUST be computed as:

`kristal_id = sha256( JCS( exchange_without_signatures ) )`

Where:
- `exchange_without_signatures` is the Exchange artifact with all signature material removed (see Section 3).

### 2.2.2 Runtime Pack identity
- Runtime Packs MUST have a stable `pack_id` or equivalent identifier.
- `pack_id` MUST be content-addressed from the Runtime Pack’s declared reproducibility surface (at minimum: manifest + referenced pack payload hashes, see Section 4).

## 2.3 Canonicalization profile declaration
Artifacts MUST declare:
- `canonicalization_profile` (e.g., `"jcs-rfc8785"`)
- `canonicalization_version` (e.g., `"1.0"`)

This enables cross-toolchain comparison and future evolution.

---

# 3. Integrity and signatures (mandatory)

## 3.1 Signature envelope
If an artifact includes signatures, it MUST do so using a clearly separated signature envelope such that:
- signature fields can be removed deterministically prior to hashing, and
- the hashed content is unambiguous.

## 3.2 Hash/sign workflow (normative)
Signing and verification MUST follow this order:

1. Remove signature material (if present)
2. Canonicalize via JCS
3. Hash (SHA-256)
4. Verify and/or attach signatures

## 3.3 Fail-closed semantics (normative)
If an artifact declares **any** of the following:
- content hash,
- signature,
- signer identity / key reference,

then verifiers MUST:
- **fail closed** if verification fails,
- **fail closed** if declared integrity material is malformed or ambiguous,
- **fail closed** if the declared hash does not match the computed hash.

Unknown non-integrity fields MUST be ignored for forward compatibility, but integrity fields are never “best-effort”.

---

# 4. Minimal reproducibility requirements (mandatory)

Kristal v3 requires reproducibility to be a first-class acceptance criterion.

## 4.1 Minimal reproducibility manifest (required fields)
Every Exchange and Runtime Pack MUST have a manifest containing at minimum:

- `build_id` (unique identifier for the build)
- `build_timestamp` (ISO 8601)
- `compiler` object:
  - `name`
  - `version`
- `config_hash` (hash of the full build configuration)
- `input_snapshots` list (content-addressed references to inputs used)
- `canonicalization_profile` + `canonicalization_version`
- `policy_selections` (portable policy selection identifiers; see `03-reproducibility/allowed-runtime-pack-policies.md`)
- declared hashes for the artifact payload(s)

## 4.2 Rebuild determinism requirement
Given:
- identical input snapshots,
- identical compiler version,
- identical configuration (as defined by `config_hash`),
- identical policy selections,

then:
- Exchange rebuild MUST produce identical `kristal_id`.
- Runtime Pack rebuild MUST produce identical `pack_id` and identical declared payload hashes.

If deterministic output cannot be achieved for some optional optimization, that optimization MUST be moved behind an optional profile or explicitly excluded from the reproducibility surface.

---

# 5. Inter-system interface contracts (mandatory)

Kristal v3 is defined as a pipeline boundary with strict contracts.

## 5.1 Claim-IR contract (proposal boundary)
- Extractors (LLMs, classical systems, hybrids) MUST output **only Claim-IR**.
- Claim-IR MUST be schema-constrained and MUST include:
  - explicit uncertainty representation,
  - explicit evidence pointers,
  - no implicit coercion into resolved entities.

Schema: `02-schemas/claim-ir.schema.json`

## 5.2 Resolution contract (SenTient boundary)
Resolution MUST output a deterministic “resolved Claim-IR” object that includes:
- ranked candidate QIDs/PIDs for entity/property surfaces,
- normalized literals with explicit type info,
- explicit representation of unresolved ambiguity,
- warnings/errors as structured, machine-readable codes.

Schema: `02-schemas/resolved-claim-ir.schema.json`

## 5.3 Validation contract (acceptance gate)
Validation MUST be deterministic and MUST:
- produce a structured validation report,
- assign stable machine-readable codes,
- block compilation if validation fails.

Schema: `02-schemas/validation-report.schema.json`

Rule: **If validation fails, compilation MUST NOT proceed.**

## 5.4 Exchange commit contract
Exchange generation MUST:
- define exactly what is included/excluded for hashing,
- define merge/incremental semantics if supported,
- preserve traceability from Exchange statements to Claim-IR and evidence.

Manifest schema: `02-schemas/exchange-manifest.schema.json`

## 5.5 Runtime Pack contract
Runtime Pack generation MUST:
- be reproducible per Section 4,
- declare all policy selections affecting query behavior and indexing,
- remain offline-executable (no network dependency required for query execution).

Manifest schema: `02-schemas/runtime-pack-manifest.schema.json`

---

# 6. Deterministic baseline exports (mandatory when implemented)

Kristal v3 defines baseline export profiles to support interoperability with Wikibase/Wikidata-shaped ecosystems.

If an implementation supports exports, it MUST support:

1. **Deterministic JSON-LD 1.1 export**
   - under a stable declared context/profile
   - byte-stable given identical inputs and policies

2. **Deterministic WDQS-compatible RDF export**
   - including a declared mapping for truthy/best-rank semantics (or explicit absence)

Export details live in:
- `05-profiles/profile-jsonld-export.md`
- `05-profiles/profile-rdf-wdqs-export.md`

---

# 7. Query semantics (core vs profile)

Runtime Packs MUST support constrained offline queries.
The detailed query contract is specified in:
- `04-query/query-contract.md`

Core requires at minimum:
- triple-pattern querying over (s, p, o) with bound/unbound fields,
- deterministic result ordering given selected ordering policy,
- deterministic paging semantics (cursor or offset) as declared.

TPF-like pagination + cardinality metadata is an optional profile:
- `05-profiles/profile-query-tpf-pagination.md`

---

# 8. Profiles and extensions

## 8.1 Profiles are explicit
Any optional capability MUST be specified as a profile.
If a profile is enabled or claimed:
- it MUST be declared in manifests,
- it MUST be testable,
- it MUST NOT redefine core identity or validation semantics.

## 8.2 RDF integrity (optional profile)
RDF Dataset Canonicalization and `rdf_hash` are OPTIONAL, due to worst-case cost.
If enabled, it MUST:
- use RDFC-1.0 canonicalization,
- gate in CI against a declared subset of tests,
- enforce resource limits, and fail closed on limit breach.

See: `05-profiles/profile-rdf-integrity-rdfc.md`

## 8.3 Provenance packaging (optional profile)
Nanopublication + PROV-O packaging is OPTIONAL.

See: `05-profiles/profile-provenance-nanopub-provo.md`

---

# 9. Compliance and test vectors (mandatory)

Implementations MUST ship and pass:
- JCS canonicalization vectors
- expected hashes for representative artifacts
- fixtures for validation pass/fail behavior
- query behavior fixtures for determinism

See: `09-test-vectors/`

---

# 10. Forward compatibility rules

- Readers MUST ignore unknown non-integrity fields.
- Readers MUST fail closed on declared integrity material that does not verify.
- Writers SHOULD avoid breaking schema changes; if unavoidable, MUST bump schema version and declare it.

---

# 11. Summary of “v3 core” obligations

A v3 core implementation MUST:
- use RFC 8785 JCS for canonical_json,
- compute content-addressed IDs from JCS with signatures excluded,
- enforce fail-closed integrity semantics,
- emit minimal reproducibility manifests,
- enforce pipeline contracts (Claim-IR, Resolution, Validation),
- produce deterministic exports when supported,
- provide test vectors and CI gating for core behaviors.
