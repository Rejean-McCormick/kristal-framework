# SenTient Resolution Contract (Kristal v3 Integration)

## Status
Draft (normative integration contract)

## Purpose
Define the required **inputs**, **outputs**, and **deterministic semantics** for SenTient when SenTient acts as the reconciliation / resolution engine in the Kristal v3 pipeline.

This contract ensures:
- ambiguity is preserved explicitly (no silent coercion),
- outputs are schema-valid and deterministically structured,
- downstream validation/compilation remains reproducible and auditable,
- failure modes are explicit and do not block the pipeline indefinitely.

This document does **not** prescribe SenTient’s internal retrieval/scoring/judging architecture. It specifies the interoperability boundary only.

## Normative language
The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are to be interpreted as normative requirements.

---

# 1. SenTient role in the Kristal pipeline

SenTient is responsible for converting Claim-IR’s surface forms and provisional values into:
- ranked candidate identifiers (QIDs/PIDs),
- normalized typed literals,
- explicit unresolved ambiguity structures,
- structured warnings/errors.

SenTient MUST NOT:
- force disambiguation without evidence,
- silently coerce ambiguous surfaces into single IDs,
- introduce new claims beyond what Claim-IR proposes.

---

# 2. Inputs (mandatory)

## 2.1 Primary input: Claim-IR batch
SenTient MUST accept as primary input:
- a **Claim-IR batch** conforming to `02-schemas/claim-ir.schema.json`.

The input MUST include (at minimum):
- `claim_id` (or stable local ID)
- surface forms for entities/properties
- proposed literals/values (where applicable)
- evidence pointers (documents/snippets/datasets)
- uncertainty representation (confidence / distribution / modality as defined by Claim-IR)

## 2.2 Optional input: hints and constraints
SenTient MAY accept optional hints, such as:
- preferred language/locale
- candidate constraints (allowed QID/PID sets)
- tenant-specific mapping overlays
- prior resolution state (for incremental resolution)

If hints are used, SenTient MUST:
- treat them as advisory,
- record their presence in the output metadata (see Section 5).

---

# 3. Outputs (mandatory)

## 3.1 Primary output: Resolved Claim-IR batch
SenTient MUST output a **Resolved Claim-IR** batch conforming to:
- `02-schemas/resolved-claim-ir.schema.json`

The output MUST preserve a stable mapping to inputs:
- every output resolution result MUST reference the originating `claim_id`.

## 3.2 Resolution results per surface
For each resolvable surface (entity surface, property surface, value normalization target), SenTient MUST provide:

### A) Ranked candidates
- `candidates[]` sorted by decreasing score
- each candidate includes:
  - `id` (QID or PID)
  - `score` (numeric, comparable)
  - `evidence` (support evidence pointers or feature justification refs)
  - optional `features` (informative)

### B) Decision state
One of the following MUST be explicitly recorded:

- `RESOLVED_SINGLE` (resolved to one best candidate)
- `RESOLVED_MULTI` (multiple candidates remain plausible; not forced)
- `UNRESOLVED` (no adequate candidate)
- `ERROR` (resolution process failed in a defined way)

### C) Selected binding (only if resolved)
If in `RESOLVED_SINGLE`, SenTient MUST provide:
- `selected.id` (QID/PID)
- `selected.score`

If in `RESOLVED_MULTI`, SenTient MUST provide:
- `selected` MUST be absent
- `candidates[]` MUST remain present and complete (at least top-K; see Section 4.2)

---

# 4. Determinism and portability requirements (mandatory)

## 4.1 Output structure determinism
SenTient output MUST be deterministic in structure:
- stable field naming and typing,
- stable sorting rules,
- stable serialization order when canonicalized (JCS is applied downstream, but structure must not be nondeterministic).

This does not require the *scores* to be identical across versions; it requires that:
- the output is well-formed,
- candidate ranking is explicit,
- ambiguity is explicit,
- and the same scoring run produces stable ordering rules.

## 4.2 Candidate list truncation (top-K)
If SenTient returns only top-K candidates, then:
- K MUST be declared in output metadata, and
- truncation MUST be consistent by surface type.

A default policy SHOULD be defined (e.g., K=20 for entities, K=10 for properties), but the exact values are implementation-defined.

## 4.3 Stable sorting rules
SenTient MUST define deterministic tie-breakers for candidates:
- primary: score descending
- tie-breaker: stable ID ordering (lexicographic QID/PID)
- final: deterministic hash of (surface, candidate ID) if needed

This avoids nondeterminism when scores collide.

## 4.4 Literal normalization determinism
When normalizing literals (dates, quantities, coordinates, identifiers), SenTient MUST:
- produce typed outputs in a deterministic schema form,
- include explicit unit/precision fields when relevant,
- record any lossy normalization decisions as warnings (see Section 6).

---

# 5. Output metadata (mandatory)

Resolved Claim-IR output MUST include metadata sufficient for downstream validation and auditing:

## 5.1 Required metadata
- `resolution_run_id` (unique)
- `sentient_version` (resolver version string)
- `timestamp` (ISO 8601)
- `input_batch_ref` (content-addressed ref to Claim-IR batch)
- `hints_used` (boolean)
- `hint_refs[]` (if hints_used=true; references to hint payloads)
- `policies` object containing (at minimum):
  - `candidate_top_k` per surface type
  - `tie_breakers` declaration
  - `normalization_ruleset_id` (versioned identifier)

## 5.2 Optional metadata
- resource limits encountered (timeouts, caps)
- corpus snapshot IDs used for candidate generation
- model identifiers used for semantic scoring (if any)

If optional metadata is recorded, it MUST NOT affect core identity unless explicitly included in the declared reproducibility surface.

---

# 6. Warnings and errors (mandatory)

## 6.1 Structured codes
SenTient MUST emit warnings/errors as:
- machine-readable `code`
- severity `{INFO, WARNING, ERROR}`
- human-readable `message`
- optional `details` payload
- `claim_id` linkage where applicable

Codes MUST be stable across minor versions.

## 6.2 Required warning categories (minimum)
SenTient SHOULD include warning codes for:
- unresolved entity/property surfaces
- ambiguous multi-candidate surfaces
- literal normalization losses (rounding, unit coercion, truncation)
- evidence insufficiency / weak grounding
- resource limit truncations (top-K truncation, timeouts)

## 6.3 Failure behavior (mandatory)
If SenTient cannot complete resolution for a claim due to transient issues (timeouts, upstream outages):
- SenTient MUST return a syntactically valid Resolved Claim-IR,
- mark affected surfaces as `UNRESOLVED` or `ERROR`,
- emit explicit error codes,
- and MUST NOT silently drop claims.

---

# 7. Preservation of ambiguity (mandatory)

SenTient MUST preserve unresolved ambiguity rather than forcing a single interpretation:
- If evidence is insufficient for a unique resolution, SenTient MUST use `RESOLVED_MULTI` or `UNRESOLVED`.
- Downstream validation MAY reject certain unresolved forms depending on rules, but SenTient must not hide ambiguity.

---

# 8. Interaction with Validation (contract coupling)

The Validation stage MUST be able to:
- deterministically accept or reject Resolved Claim-IR based on declared rules.

Therefore SenTient MUST:
- keep all resolution states explicit,
- keep all warning/error codes stable and structured,
- provide all required metadata needed to interpret results.

---

# 9. Security and multi-tenancy (mandatory where applicable)

## 9.1 Tenant scoping
If SenTient is used in a multi-tenant environment:
- tenant-specific hint overlays MUST be isolated by tenant_id,
- output metadata MUST indicate tenant scope if applicable.

## 9.2 Input confidentiality
SenTient MUST NOT leak evidence pointers or internal corpora references across tenants.

(Operational enforcement is system-specific; this is a boundary requirement.)

---

# 10. Conformance checklist (SenTient)

SenTient satisfies this contract if it:
- accepts Claim-IR and produces schema-valid Resolved Claim-IR,
- emits ranked candidates and explicit decision states,
- preserves ambiguity (no silent coercion),
- applies deterministic sorting and tie-breakers,
- produces deterministic literal normalization structures,
- emits structured warnings/errors with stable codes,
- returns valid outputs even on partial failure (no dropped claims).
