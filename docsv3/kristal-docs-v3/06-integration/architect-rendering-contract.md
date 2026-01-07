# Architect rendering contract (v3 integration)

## Status

Draft (v3 integration contract)

## Purpose

This document defines the **contract between Architect and Kristal v3**.

Architect is the **deterministic renderer**: it produces text (and other publishable outputs) **only from validated Kristal knowledge**, without introducing new facts, and with **complete traceability** from every rendered assertion back to Kristal claim(s) and evidence.

This contract applies to:

* rendering from **Runtime Pack query results**, or
* rendering from **Exchange-derived validated query results** (where the query output is itself a deterministic, signed/verified artifact).

Normative keywords: MUST, MUST NOT, SHOULD, SHOULD NOT, MAY.

---

## 1. Scope and non-goals

### 1.1 In scope

* Inputs Architect is allowed to consume
* Determinism requirements for rendering
* Prohibitions on introducing new facts
* Traceability requirements (claim/evidence mapping)
* Error and refusal behavior
* Multilingual rendering rules and templating boundaries
* Output packaging (text + machine-readable trace map)

### 1.2 Out of scope

* How Kristals are built (Orgo/SenTient pipeline)
* Runtime query engine implementation details
* UI presentation requirements in Konnaxion
* Editorial policy (tone/style) beyond determinism and factuality constraints

---

## 2. Contract overview

Architect rendering is a **pure function** over validated inputs:

* Inputs: validated knowledge + rendering spec
* Output: deterministic text + trace metadata
* No network access required to be correct
* No use of probabilistic sources as factual inputs during rendering

---

## 3. Inputs (mandatory constraints)

### 3.1 Allowed inputs

Architect MUST accept only one of the following input types:

**A) Runtime Pack query result bundle**

* `pack_id` (or pack content ID)
* `pack_manifest` (or manifest reference)
* `query_contract_version`
* `query_results` (ordered results)
* `result_provenance` (optional but recommended: query parameters, cursor chain)

**B) Exchange-derived validated query bundle**

* `source_kristal_id`
* `exchange_manifest` (or reference)
* deterministic query spec
* `query_results` (ordered results)
* validation proof that the results were computed from the referenced Exchange under a declared query contract

Architect MUST reject inputs that:

* are not validation-attested, or
* do not include stable IDs for claims/statements, or
* cannot be traced back to a specific `kristal_id` / pack id.

### 3.2 Required identifiers in input

Inputs MUST include enough identifiers to support traceability:

* `kristal_id` (Exchange) or `pack_id` (Runtime Pack)
* stable identifiers for each asserted fact (preferred: `statement_id`; otherwise a deterministic surrogate)
* stable evidence pointer(s) for each asserted fact (`evidence_id`, `reference_id`, or equivalent)

### 3.3 Rendering request specification (required)

Architect MUST take a rendering request object that includes:

* `render_kind` (e.g., `"article"`, `"snippet"`, `"summary"`, `"qa"`, `"card"`)
* `language` (BCP-47 tag, e.g., `en`, `fr-CA`)
* `audience_profile` (optional; must not change factual content)
* `template_id` (or `render_profile_id`) and version
* `projection` (e.g., `"full"` or `"truthy"`, if supported)
* `constraints` (see Section 4)

---

## 4. Deterministic rendering rules (mandatory)

### 4.1 Determinism requirement

For identical:

* validated input bundle bytes,
* template/profile id + version,
* language,
* rendering parameters,

Architect MUST produce identical outputs (text + trace map), modulo explicitly-declared nondeterministic fields (e.g., timestamps) that are not included in output hashing.

### 4.2 No new facts (hard constraint)

Architect MUST NOT introduce any factual assertion that is not supported by at least one input claim/statement.

This includes:

* numbers, dates, names, causal relationships, superlatives, and categorical claims
* “common knowledge” filler if it asserts new facts (not allowed)
* external enrichment from the network (not allowed for factual claims)

Architect MAY add:

* purely structural text (headings, transitions)
* clearly-marked uncertainty language **only if** the uncertainty is present in the input (e.g., qualifiers, confidence, ambiguity flags)
* non-factual stylistic phrasing that does not add information

### 4.3 Ambiguity preservation

If input contains unresolved ambiguity (multiple candidates, uncertain values), Architect MUST either:

* render the ambiguity explicitly (e.g., “may refer to …”), or
* refuse to render the ambiguous claim as a fact (and record a refusal reason)

Architect MUST NOT silently pick one disambiguation unless the input explicitly declares a resolved selection.

### 4.4 Projection consistency

If the input bundle declares a projection (e.g., `truthy`), Architect MUST:

* render only from that projection, and
* label the output metadata with the projection used.

---

## 5. Output requirements (mandatory)

Architect MUST output a **render bundle** with:

1. `rendered_text` (or structured blocks)
2. `trace_map` (machine-readable)
3. `render_metadata`

### 5.1 render_metadata (required fields)

* `render_kind`
* `language`
* `template_id` and `template_version`
* `source_kristal_id` or `pack_id`
* `query_contract_version` (if from runtime)
* `projection`
* `build_id` (Architect run identifier)
* `created` (timestamp)

### 5.2 trace_map (required fields)

The trace map MUST provide **complete coverage** of factual assertions.

Minimum fields:

* `segments[]`: list of rendered segments with stable segment ids

  * `segment_id`
  * `text_span` (byte offsets or token indices) OR `block_id` (if structured blocks)
  * `assertions[]`

Each assertion MUST include:

* `assertion_id` (stable within the render)
* `support[]`: list of support pointers, each containing:

  * `statement_id` (preferred) OR a deterministic statement pointer (subject/property/value + hashes)
  * `evidence_ids[]` (or reference pointers)
  * `source` (Exchange vs Runtime Pack)
* `confidence` (optional; only if present in input)
* `notes` (optional; must not introduce facts)

### 5.3 Coverage rule (mandatory)

Every factual statement in `rendered_text` MUST have at least one `support[]` entry in `trace_map`.

If a segment cannot be supported, Architect MUST:

* either omit the claim from output, or
* include it only as an explicitly-marked uncertainty statement with trace back to the uncertainty-bearing input, or
* fail the render with a deterministic error.

---

## 6. Validation and refusal behavior (mandatory)

Architect MUST implement deterministic refusal/error codes, including at minimum:

* `UNVERIFIED_INPUT` (input not validated / cannot prove provenance)
* `MISSING_TRACE_IDS` (no stable statement/evidence pointers)
* `AMBIGUOUS_INPUT` (ambiguity cannot be rendered under requested constraints)
* `UNSUPPORTED_RENDER_KIND`
* `PROJECTION_MISMATCH`
* `POLICY_VIOLATION_NEW_FACT_RISK` (attempted output would introduce unsupported fact)

Architect MUST return:

* `status = "ok" | "refused" | "error"`
* `code` (stable string)
* `message` (human-readable)
* `details` (optional structured fields)

---

## 7. Multilingual rendering (mandatory constraints)

### 7.1 Language fidelity

Architect MAY translate labels and explanatory text, but MUST NOT alter factual content.

### 7.2 Localized formatting

Architect MAY apply locale-specific formatting (numbers, dates) **only if** it does not change meaning and the underlying normalized value remains traceable.

### 7.3 Determinism across languages

Rendering in different languages may produce different bytes, but each language output MUST be deterministic given the same inputs and the same template/profile.

---

## 8. Security and offline constraints

* Architect MUST NOT require network calls to produce correct factual output.
* If network calls are used for non-factual assets (fonts, templates), they MUST NOT affect factual assertions.
* If inputs are signed, Architect SHOULD verify signatures (or rely on Orgo/Konnaxion verified delivery), but MUST reject inputs that cannot be proven validated under the deployment’s policy.

---

## 9. Conformance tests (required)

An implementation claiming conformance to this contract MUST provide tests for:

* determinism: same input yields identical render bundle
* trace coverage: every asserted claim has support pointers
* “no new facts”: attempts to inject unsupported facts are refused/error
* ambiguity: ambiguous inputs are rendered explicitly or refused deterministically
* projection handling: truthy/full mismatch triggers deterministic error/refusal

---

## 10. Open questions

* Do we standardize the `trace_map` format as JSON Schema in `02-schemas/` (recommended)?
* Do we require `statement_id` in v3 core, or allow only deterministic pointers until statement_id is mandatory?
* Should Architect output itself be content-addressed and optionally signed as a derived artifact (likely a profile)?
