# Errata and alignment (Kristal v4 docs)

## Status
Editorial (v3 docs). Normative for documentation consistency and examples.

## Purpose
This document records the **alignment decisions** required to keep Kristal v4 prose, schemas, and examples consistent. It is the single place to resolve or track mismatches discovered during v3 doc updates.

## Applies to
- `01-core-spec/*`
- `02-schemas/*`
- `03-reproducibility/*`
- `10-examples/*`
- `00-overview/*` (when it defines artifact contracts)

---

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
- Exclude the output ID field itself (e.g., `kristal_id`, `runtime_pack_id`, `federation_id`, `shard_id`).
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
All JSON Schemas MUST use the `kristal.org` namespace.

### G) Identifier formats (registry IDs vs artifact IDs)
To avoid schema/example mismatches:
- `federation_id`, `shard_id`, `kristal_id` MUST use the canonical `sha256:<hex>` form unless a specific artifact explicitly standardizes a different namespace.
- Policy artifacts (e.g., Authority Registry) MAY use a richer namespaced ID (e.g., `kristal:authority-registry:sha256:<hex>`).

**Schema rule:** if an example uses a namespaced ID, schemas MUST accept it (do not over-constrain to `sha256:<hex>` only).

### H) “additionalProperties: false” and examples
If a schema sets `additionalProperties: false`, then every top-level field present in `10-examples/*.json` MUST be declared in that schema’s `properties`. No “extra fields” in examples.

### I) Federation manifests: registry + publisher metadata
Federation examples use:
- `authority_registry_ref` (pointer + hash for pinned policy),
- `publisher` (identity metadata),

Therefore, the Federation Manifest schema MUST declare these fields if examples keep them at top-level (Decision H).

---

## Errata list and resolutions

### 1) Canonicalization profile mismatch across docs
**Resolution**
Adopt Section A above. Update prose, schemas, and examples.

### 2) “Minimal reproducibility manifest” prose does not match schemas
**Resolution**
- Treat JSON schemas in `02-schemas/` as the source of truth for field names and structure.
- Update core prose to match schema names/groupings.

**Mapping table (legacy → schema canonical)**
| Legacy prose term | Canonical schema field |
|---|---|
| `build_timestamp` | `created_at` |
| `compiler {name, version}` | `build.compiler_name`, `build.compiler_version` (Exchange) / `compiler.name`, `compiler.version` (Runtime Pack) |
| `input_snapshots` | `inputs.*` (Exchange) |
| `policy_selections` | `policies.*` |

### 3) `alg` vs `algo` inconsistency
**Resolution**
Decision D applies: standardize on `alg`.

### 4) Ambiguity: are timestamps part of hashed material?
**Resolution**
Decision C applies: timestamps are not part of any hashed-material projection unless explicitly stated by a profile (no v3 core profile does).

### 5) Ambiguity: signature placement and hashing exclusions
**Resolution**
Decision B applies:
- Signatures/attestations are excluded from hash targets wherever they appear.
- Canonical placement for manifests/examples is a top-level `signatures[]` array only.

### 6) Shard schema/example swapped
**Symptom**
`02-schemas/exchange-shard-manifest.schema.json` contains an instance-like object; `10-examples/exchange-shard-manifest.example.json` contains a JSON Schema.

**Resolution**
Swap contents (or rename) so:
- `02-schemas/*.schema.json` are JSON Schemas
- `10-examples/*.example.json` are instances

### 7) Federation example contains fields not declared in schema
**Symptom**
Federation example includes `authority_registry_ref` and `publisher`, but the schema rejects unknown properties due to `additionalProperties:false`.

**Resolution**
Apply Decisions H + I:
- Either declare both fields in schema (recommended), or move them under `extensions` in the example and remove from top-level.

### 8) Registry ID pattern mismatch (sha256 vs kristal:* namespaced)
**Symptom**
Authority Registry IDs may be namespaced (e.g., `kristal:authority-registry:sha256:...`) while federation examples may use `sha256:...`.

**Resolution**
Decision G applies:
- Do not constrain registry IDs to a single pattern in `authority_registry_ref.registry_id`.
- Accept both `sha256:<hex>` and `kristal:*:sha256:<hex>`.

---

## Required file updates (tracking)

### Core determinism + hashing rules
- `03-reproducibility/deterministic-build-rules.md`
  - Update canonicalization requirements to the canonical pair.
  - Clarify hash target exclusions and timestamp non-participation.

### Runtime Pack Manifest
- `02-schemas/runtime-pack-manifest.schema.json`
  - `$id` must be `https://kristal.org/schemas/v3/runtime-pack-manifest.schema.json`
  - Constrain `build.canonicalization_profile/version` to canonical values
  - Ensure signature shape matches Decision E
- `10-examples/runtime-pack-manifest.example.json`
  - Must match schema + Decisions A–E.

### Exchange Manifest + Exchange example
- `02-schemas/exchange-manifest.schema.json`
  - Standardize hash algorithm key to `alg`
  - Ensure signature object matches Decision E
- `10-examples/exchange.example.json`
  - Replace any `algo` → `alg`
  - Ensure canonicalization fields match Decision A

### Sharding & Federation
- `02-schemas/exchange-shard-manifest.schema.json` + `10-examples/exchange-shard-manifest.example.json`
  - Fix swap (Errata #6)
- `02-schemas/exchange-federation-manifest.schema.json` + `10-examples/exchange-federation-manifest.example.json`
  - Add/align `authority_registry_ref` + `publisher` (Errata #7)
  - Relax `authority_registry_ref.registry_id` pattern (Errata #8)

### Authority Registry + Revocations
- `02-schemas/authority-registry.schema.json` + `10-examples/authority-registry.example.json`
  - Ensure pinned/versioned policy data is self-consistent
- `02-schemas/revocations.schema.json` + `10-examples/revocations.example.json`
  - Standardize revocation artifact format (optional but recommended for offline verification)

---

## Compatibility guidance (non-normative)
Implementations MAY accept legacy spellings during a transition window, but:
- Emitters MUST write canonical field names and canonicalization identifiers.
- Validators SHOULD warn on legacy forms and recommend migration.

---

## Change log
- 2026-02-26: initial alignment decisions captured (canonicalization, hash exclusions, alg/algo, timestamps, signature placement, schema $id namespace).
- 2026-02-26: added federation/sharding alignment rules (schema/example swap, federation schema fields, registry ID pattern acceptance).