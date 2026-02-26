# Errata and alignment (Kristal v3 docs)

## Status
Editorial (v3 docs). Normative for documentation consistency and examples.

## Purpose
This document records the **alignment decisions** required to keep Kristal v3 prose, schemas, and examples consistent. It is the single place to resolve or track mismatches discovered during v3 doc updates.

## Applies to
- `01-core-spec/*`
- `02-schemas/*`
- `03-reproducibility/*`
- `10-examples/*`

## Canonical conventions (must be applied everywhere)

### A) Canonicalization profile (v3 core)
**Canonical values**
- `canonicalization_profile = "kristal.v3:jcs-rfc8785"`
- `canonicalization_version = "1"`

**Where recorded**
- Exchange artifact (top-level fields)
- Runtime Pack Manifest (under `build.*`)

**Deprecations**
The following legacy spellings are deprecated and must not appear in updated specs/examples:
- `canonicalization_profile: "jcs-rfc8785"`
- `canonicalization_profile: "jcs-rfc8785@1"`
- `canonicalization_version: "RFC8785"`
- `canonicalization_version: "1.0"`

### B) Hash target exclusions (IDs and hashes)
When computing any content-addressed ID:
- Exclude the output ID field itself (e.g., `kristal_id`, `runtime_pack_id`).
- Exclude all signature/attestation material (`signatures`, and any equivalent fields), regardless of location.

### C) Timestamps vs determinism
- `created_at` MAY reflect real build time.
- `created_at` MUST NOT affect any content-addressed IDs.

### D) Hash object field names (alg vs algo)
**Decision:** use `alg` as the field name for algorithm identifiers in hash objects, consistently.
- Replace any `algo` occurrences in schemas/examples with `alg`.

(If an artifact currently uses `algo`, consumers MAY accept it for backward compatibility, but emitters MUST write `alg`.)

### E) Signature object shape (schemas + examples)
**Decision:** manifests and examples use a minimal signature object:
- `key_id` (string)
- `alg` (string)
- `signature` (string)
- `created_at` (RFC3339 string, optional but recommended)

Any richer signature payload/envelope belongs to an explicit signing profile and must not be required for v3 core conformance.

### F) Schema identifiers ($id)
All JSON Schemas MUST use the `kristal.org` namespace, not `example.org`.

---

## Errata list and resolutions

### 1) Canonicalization profile mismatch across docs
**Symptom**
Different documents/examples use different identifiers/versions for the same canonicalization scheme.

**Resolution**
Adopt Section “Canonicalization profile (v3 core)” above. Update:
- Core prose references
- Schemas that constrain canonicalization fields
- Examples/test vectors

### 2) “Minimal reproducibility manifest” prose does not match schemas
**Symptom**
The prose lists fields that do not align with the Exchange manifest schema structure/names.

**Resolution**
- Treat JSON schemas in `02-schemas/` as the source of truth for field names and structure.
- Update the core prose to match schema names and groupings.

**Mapping table (legacy → schema canonical)**
| Legacy prose term | Canonical schema field |
|---|---|
| `build_timestamp` | `created_at` |
| `compiler {name, version}` | `build.compiler_name`, `build.compiler_version` (Exchange) / `compiler.name`, `compiler.version` (Runtime Pack) |
| `input_snapshots` | `inputs.*` (Exchange) |
| `policy_selections` | `policies.*` |

(Extend this table as additional mismatches are found.)

### 3) `alg` vs `algo` inconsistency
**Symptom**
Some schemas/examples use `algo`, others use `alg`.

**Resolution**
Decision D applies: standardize on `alg`.

### 4) Ambiguity: are timestamps part of hashed material?
**Symptom**
Some prose implies manifest fields (including timestamps) are hashed, which contradicts deterministic ID requirements.

**Resolution**
Decision C applies: timestamps are not part of any hashed-material projection unless explicitly stated by a profile (no v3 core profile does).

### 5) Ambiguity: signature placement and hashing exclusions
**Symptom**
Docs leave room for interpretation about where signatures can appear and what gets removed before hashing.

**Resolution**
Decision B applies:
- Signatures/attestations are excluded from hash targets wherever they appear.
- Canonical placement for manifests/examples is a top-level `signatures[]` array only.

---

## Required file updates (tracking)

### Deterministic build rules
- Update canonicalization requirements to the canonical pair.
- Clarify hash target exclusions and timestamp non-participation.
- Ensure runtime pack hashed-material projection requirements are consistent with the Runtime Pack Manifest schema.

### Runtime Pack Manifest schema
- `$id` must be `https://kristal.org/schemas/v3/runtime-pack-manifest.schema.json`
- Add `build.canonicalization_version`
- Constrain `build.canonicalization_profile/version` to canonical values
- Ensure signature shape matches Decision E

### Exchange Manifest schema + Exchange example
- Standardize hash algorithm key to `alg`
- Standardize canonicalization fields to canonical values
- Ensure any signature object matches Decision E
- Ensure any `kristal_id` example reflects the canonical ID format used by the IDs spec

---

## Compatibility guidance (non-normative)
Implementations MAY accept legacy spellings during a transition window, but:
- Emitters MUST write canonical field names and canonicalization identifiers.
- Validators SHOULD warn on legacy forms and recommend migration.

---

## Change log
- 2026-02-26: initial alignment decisions captured (canonicalization, hash exclusions, alg/algo, timestamps, signature placement, schema $id namespace).