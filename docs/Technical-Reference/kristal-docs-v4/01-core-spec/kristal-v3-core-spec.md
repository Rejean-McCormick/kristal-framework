# Kristal v4 Core Specification

## Status
Draft (normative core)

## Purpose
Define the **minimal normative requirements** for Kristal v4 conformance, with a strong focus on:
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
An implementation is **Kristal v4 Core Conformant** if it:
1. Implements the **canonicalization + identity** rules in this document.
2. Implements **fail-closed integrity verification** semantics.
3. Emits and honors **schema-conformant manifests** that satisfy minimal reproducibility requirements.
4. Enforces the **inter-system interface contracts** (schemas + deterministic rules).
5. Produces **deterministic baseline exports** when enabled.

Optional profiles do not affect core conformance unless the implementation claims support for them. If an implementation claims a profile, it MUST satisfy that profile’s requirements.

---

# 1. Artifact model

Kristal v4 defines two primary artifact classes:

1. **Kristal Exchange Artifact (Exchange)**
   - Canonical, auditable, content-addressed source of truth.
   - Mergeable representation of validated knowledge claims.

2. **Kristal Runtime Pack (Runtime Pack)**
   - Derived, indexed, offline-executable representation.
   - Constrained query semantics; NOT full SPARQL.

Both artifact classes MUST have an associated **Manifest** (schema-conformant; see Section 4).

---

# 2. Canonicalization and identity (mandatory)

## 2.1 Canonical JSON
- `canonical_json` MUST be defined as **RFC 8785 JSON Canonicalization Scheme (JCS)**.
- Implementations MUST NOT introduce additional canonicalization steps that change byte output beyond JCS.

## 2.2 Content-addressed IDs

### 2.2.1 `kristal_id` for Exchange (mandatory)
The Exchange `kristal_id` MUST be derived from a well-defined **hash target** and encoded in a portable string form.

**Hash target (mandatory):**
To compute IDs, implementations MUST derive a hash target object `T` from the Exchange artifact `E` by removing:
1. `kristal_id` (the output field), and
2. any declared signatures/attestations fields (wherever they appear).

**Computation (mandatory):**
1. Produce `T = hash_target(E)` as above.
2. Serialize `T` to bytes `B = JCS(T)` using RFC 8785.
3. Compute `H = SHA-256(B)`.
4. Encode `H` as lowercase hex `hex(H)`.
5. Set:
   - `kristal_id = "sha256:" + hex(H)`.

**Relationship to Exchange Manifest (mandatory when manifest present):**
If an Exchange Manifest is present:
- `content_hash.alg` MUST be `"sha256"`.
- `content_hash.value` MUST equal `hex(H)` computed from the Exchange hash target.
- `kristal_id` MUST equal `"sha256:" + content_hash.value`.

**Determinism constraint (mandatory):**
Any field that remains inside the Exchange hash target MUST NOT be populated from non-deterministic sources (wall-clock time, nondeterministic randomness, nondeterministic iteration order). If timestamps are present inside the Exchange artifact, they MUST be derived deterministically from inputs/policies (not “build time now”).

### 2.2.2 Runtime Pack identity (mandatory)
Runtime Packs MUST have a stable, content-addressed identifier.

- A Runtime Pack MUST declare `runtime_pack_id` (sha256 hex) in its manifest schema.
- Other integration surfaces MAY refer to this same identifier as `pack_id`; unless a profile says otherwise, `pack_id` is a synonym for `runtime_pack_id`.

**Derivation rule (mandatory):**
`runtime_pack_id` MUST be derived from a canonical, deterministic representation of the pack’s hashed material, and MUST be test-vectorized.

At minimum, the hashed material MUST include:
- the Runtime Pack Manifest content **excluding signatures**, OR a canonical projection of it that also excludes non-deterministic fields that MUST NOT affect IDs (e.g., `created_at`),
- the file inventory with each file’s `sha256` and relative `path`,
- the referenced Exchange ID.

The exact hashed-material definition MUST be documented and test-vectorized.

## 2.3 Canonicalization profile declaration (mandatory)
Artifacts and manifests MUST declare which canonicalization profile governs any hashed JSON bytes.

### 2.3.1 Split form (preferred, mandatory where the schema provides both fields)
Where available, producers MUST declare:
- `canonicalization_profile` (e.g., `"jcs-rfc8785"`)
- `canonicalization_version` (e.g., `"1"`)

### 2.3.2 Combined identifier form (allowed where only one field exists)
Some schemas may record a combined identifier in a single field (e.g., `build.canonicalization_profile = "jcs-rfc8785@1"`).
Where this occurs, the producer MUST ensure it is parseable as:

`canonicalization_id = canonicalization_profile + "@" + canonicalization_version`

Consumers MUST treat the combined form and the split form as equivalent after normalization.

---

# 3. Integrity and signatures (mandatory)

## 3.1 Signature envelope (mandatory)
If an artifact or manifest includes signatures, it MUST do so using a clearly separated signature envelope such that:
- signature fields can be removed deterministically prior to hashing, and
- the hashed content is unambiguous.

**Location (mandatory):**
Signatures MUST be placed in a top-level `signatures` array (for the relevant artifact or manifest), not interleaved into hashed content.

## 3.2 Hash/sign workflow (normative)
Signing and verification MUST follow this order:

1. Remove signature material (if present)
2. Canonicalize via the declared canonicalization profile (JCS for core)
3. Hash (SHA-256 for core IDs and core-declared integrity hashes)
4. Verify and/or attach signatures

## 3.3 Fail-closed semantics (normative)
If an artifact or manifest declares **any** integrity material (including any of the following):
- content hash / declared file hashes,
- signature(s),
- signer identity / key reference,

then verifiers MUST:
- **fail closed** if verification fails,
- **fail closed** if declared integrity material is malformed, incomplete, or ambiguous,
- **fail closed** if any declared hash does not match the computed hash.

Unknown non-integrity fields MUST be ignored for forward compatibility, but integrity fields are never “best-effort”.

---

# 4. Minimal reproducibility requirements (mandatory)

Kristal v4 requires reproducibility to be a first-class acceptance criterion.

## 4.1 Manifests are schema-conformant (mandatory)
Exchange and Runtime Pack manifests MUST conform to their schemas:

- Exchange Manifest: `02-schemas/exchange-manifest.schema.json`
- Runtime Pack Manifest: `02-schemas/runtime-pack-manifest.schema.json`

### 4.1.1 Exchange Manifest minimal required fields (mandatory)
At minimum, the Exchange Manifest MUST include the schema-required fields:
- `schema_version`
- `kristal_id`
- `canonicalization_profile`
- `canonicalization_version`
- `content_hash` (object with `alg`, `value`)
- `build` (must include `build_id`, `compiler_name`, `compiler_version`, `config_hash`)
- `inputs` (must include `source_snapshot_id`)

### 4.1.2 Runtime Pack Manifest minimal required fields (mandatory)
At minimum, the Runtime Pack Manifest MUST include the schema-required fields:
- `manifest_version`
- `runtime_pack_id`
- `created_at`
- `exchange_ref`
- `compiler` (must include `name`, `version`)
- `build` (must include `build_id`, `deterministic=true`, `canonicalization_profile`, `config_hash`)
- `policies` (schema-defined policy objects)
- `files[]` (each file entry must include `path`, `role`, `sha256`, `size_bytes`)

**File inventory rule (mandatory):**
The manifest’s file inventory and hashes MUST be sufficient for consumers to verify the pack offline.

## 4.2 Rebuild determinism requirement (mandatory)
Given:
- identical input snapshots (as referenced by manifest inputs / exchange_ref),
- identical compiler version,
- identical configuration (as defined by `config_hash`),
- identical policy selections (as recorded in manifests),

then:
- Exchange rebuild MUST produce identical `kristal_id`.
- Runtime Pack rebuild MUST produce identical `runtime_pack_id` and identical declared payload/file hashes.

If deterministic output cannot be achieved for some optional optimization, that optimization MUST be moved behind an optional profile or explicitly excluded from the reproducibility surface (and that exclusion MUST be declared and testable).

---

# 5. Inter-system interface contracts (mandatory)

Kristal v4 is defined as a pipeline boundary with strict contracts.

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

Kristal v4 defines baseline export profiles to support interoperability with Wikibase/Wikidata-shaped ecosystems.

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
- use RFC 8785 JCS for canonical JSON of hashed JSON objects,
- compute Exchange `kristal_id` from the JCS hash target with signatures excluded, encoded as `sha256:<hex>`,
- enforce fail-closed integrity semantics,
- emit schema-conformant Exchange and Runtime Pack manifests that satisfy minimal reproducibility requirements,
- enforce pipeline contracts (Claim-IR, Resolution, Validation),
- produce deterministic exports when supported,
- provide test vectors and CI gating for core behaviors.