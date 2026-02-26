# Query contract

## Status

Draft (v3)

## Purpose

This document defines the **offline query contract** for Kristal v3 **Runtime Packs** (and any local wrapper service that exposes them). The goal is a **portable, deterministic, offline-executable** query surface that supports common retrieval needs (search/navigation primitives, constrained graph lookups) without requiring full SPARQL.

This contract is intentionally constrained to preserve offline predictability and reproducibility.

---

## 1. Scope and non-goals

### 1.1 In scope

* A **triple-pattern** query model over `(s, p, o)` with any subset bound/unbound.
* Deterministic result ordering (derived from recorded pack policies).
* Deterministic paging (cursor or offset), as declared by the pack’s recorded policies.
* Deterministic failure behavior for resource limits (join caps, page caps, timeouts).
* Optional profiles: pagination envelope + optional cardinality metadata.

### 1.2 Out of scope (non-goals)

* Full SPARQL 1.1 semantics (OPTIONAL, UNION, FILTER with full expression semantics, property paths, subqueries, federated queries).
* Network dependencies: queries MUST be executable offline over a local Runtime Pack.
* Non-deterministic query behavior (randomization, adaptive sampling without declaring a profile).

---

## 2. Terminology

* **Term**: a subject/object identifier (e.g., QID-like) or literal.
* **Triple**: `(s, p, o)` where `s` and `p` are identifiers; `o` is identifier or literal.
* **Triple pattern**: a triple with any of `s`, `p`, `o` possibly unbound.
* **Binding**: a concrete value for an unbound position.
* **Result set**: ordered list of matching triples or bindings.
* **Paging mode**: either `CURSOR` or `OFFSET`, as recorded in pack policies.
* **Cursor**: opaque paging token that resumes iteration deterministically.
* **Offset paging**: deterministic paging by `(offset, limit)` against a deterministic ordering.
* **Page size**: the number of results requested per page. This is `limit` in the core contract; profiles MAY name it `page_size`.
* **join1**: a constrained two-stage lookup pattern (see Section 6) with a declared cap and strictness mode.

Normative keywords: MUST, MUST NOT, SHOULD, SHOULD NOT, MAY.

---

## 3. Query model (mandatory)

### 3.1 Supported query primitive

An implementation MUST support querying a single triple pattern:

* Inputs:
  * `s` (bound or unbound)
  * `p` (bound or unbound)
  * `o` (bound or unbound)
  * page size parameter:
    * `limit` (positive integer; core contract name), and/or
    * `page_size` (positive integer; if a pagination profile requires it)
  * paging parameters (cursor or offset mode, see Section 7)

* Output:
  * an ordered sequence of results, each result containing either:
    * matching triples `(s, p, o)`; or
    * bindings for unbound positions (`?s`, `?p`, `?o`) plus (optionally) the full triple

Implementations MUST document which output shape they use and MUST keep it stable within a declared contract identifier (see Section 11).

### 3.2 Determinism

Given:

* identical Runtime Pack bytes,
* identical recorded portable policy selections affecting ordering and query behavior,
* identical query inputs,

the implementation MUST return:

* identical result ordering, and
* identical paging behavior.

---

## 4. Data model constraints (mandatory)

### 4.1 Identifiers and literals

* Subjects and properties MUST be represented in a stable identifier space (e.g., QID/PID-like strings or an equivalent stable encoding).
* Literal normalization rules (numbers, dates, language tags) MUST be deterministic and MUST be consistent with the Runtime Pack’s manifest (or a referenced normalization profile).

### 4.2 Rank / “truthy” semantics

Runtime Packs MUST define (in their manifest or an explicitly referenced policy/profile) whether the runtime query surface:

* returns the full statement set, or
* returns a “truthy/best-rank” projection, or
* supports both as named projections.

If “truthy” is supported, the mapping rules MUST be deterministic and documented. If only exports define truthy behavior and runtime does not, the runtime MUST state that clearly.

---

## 5. Result ordering (mandatory)

### 5.1 Ordering source of truth

Result ordering MUST be derived from the Runtime Pack’s recorded ordering policy (e.g., `policies.data_ordering`). Implementations MUST NOT use implementation-dependent iteration order.

### 5.2 Stable tie-breakers

If multiple rows compare equal under the primary ordering keys, implementations MUST define stable tie-breakers (e.g., internal row-id, claim_id) so ordering is total and deterministic.

### 5.3 Ordering disclosure

Implementations MUST document:

* the primary ordering key order,
* tie-breakers,
* whether ordering is over triples, bindings, or both.

---

## 6. Resource limits and join caps (mandatory)

### 6.1 Required limits to declare

Implementations MUST declare, at minimum:

* a maximum allowed page size (`limit_max`), and
* a join cap policy if join1 is supported:
  * `join1_cap.default` (integer), and
  * `join1_cap.mode` in `{STRICT, TRUNCATE}`

These MUST be surfaced in either:

* the Runtime Pack manifest (preferably under recorded policies), or
* the query wrapper configuration (if the wrapper is the enforcement point),

and MUST remain deterministic for a given pack + configuration.

### 6.2 join1 (optional capability, but deterministic if present)

If the runtime supports a two-stage lookup (common for navigation), it MUST define join1 explicitly, for example:

1. Evaluate pattern A to produce intermediate bindings (bounded by `join1_cap.default`)
2. Use each binding to evaluate pattern B
3. Combine results deterministically

If join1 is not supported, the implementation MUST reject such requests deterministically.

### 6.3 Cap breach behavior (mandatory to define)

When a cap is exceeded, behavior MUST be determined by the declared `join1_cap.mode`:

* **STRICT**: return an error response with a stable error code (recommended for correctness)
* **TRUNCATE**: truncate intermediate results deterministically and return a `truncated=true` flag plus deterministic counts (where feasible)

Silently truncating without a flag MUST NOT occur.

---

## 7. Paging contract (mandatory)

An implementation MUST support exactly one paging mode, determined by the pack’s recorded paging policy (e.g., `policies.paging`):

* `CURSOR`, or
* `OFFSET`

### 7.1 Cursor paging (recommended)

If cursor paging is supported:

* The cursor MUST be opaque to callers.
* The cursor MUST resume iteration deterministically for the same pack and query inputs.
* The cursor MUST become invalid if:
  * the pack changes, or
  * the query parameters change (other than page size), unless explicitly supported.

Core required response fields:

* `next_cursor` (string or null)
* `has_more` (boolean)

If a pagination profile is enabled, additional envelope fields and hashing requirements are defined by that profile (see Section 10).

### 7.2 Offset paging (allowed)

If offset paging is supported:

* The implementation MUST define ordering (Section 5) such that `(offset, limit)` is deterministic.
* Responses SHOULD include:
  * `offset`
  * `limit`
  * `total` (optional; if expensive, omit or move to a cardinality profile)

---

## 8. Error model (mandatory)

### 8.1 Stable error codes

Implementations MUST return stable, machine-readable error codes, at minimum for:

* invalid query schema / parameter types
* unsupported query shape
* cap exceeded (for `STRICT` mode)
* cursor invalid/expired
* internal corruption / integrity failure (if pack verification fails)

### 8.2 Deterministic error behavior

Errors MUST be deterministic for a given:

* pack,
* query input,
* recorded policies and enforcement configuration.

---

## 9. Optional capability: cardinality metadata

If the pack declares cardinality estimates as supported (e.g., `query_contract.supports_cardinality_estimates = true`), responses MAY include:

* `cardinality` object with:
  * `type`: `"exact"` or `"approx"`
  * `value`: integer
  * `bounds`: optional `{ "lower": int, "upper": int }`
  * `confidence`: optional float in `[0,1]`
  * `scope`: e.g., `"pattern"` or `"pattern+projection"`

Rules:

* If `type="exact"`, `value` MUST be correct.
* If `type="approx"`, the implementation MUST document approximation method and what the bounds/confidence mean.

---

## 10. Optional profile: pagination envelope (TPF-like pagination)

If the pack claims `profile-query-tpf-pagination@1`, the normative request/response envelope, stable-order requirements for pagination, and (optional) cardinality fields are defined by:

* `05-profiles/profile-query-tpf-pagination.md`

This contract does not define a transport. A local wrapper MAY expose HTTP endpoints, but MUST preserve the profile’s normative behaviors and MUST remain offline.

---

## 11. Manifest requirements (mandatory for declared support)

### 11.1 Contract declaration

A Runtime Pack that claims conformance to this query contract SHOULD declare `query_contract` in the Runtime Pack Manifest:

* `query_contract.contract_id` (non-empty, versioned identifier)

If the pack claims a pagination profile, it MUST also declare:

* `query_contract.supports_pagination = true`

If the pack supports cardinality metadata, it SHOULD declare:

* `query_contract.supports_cardinality_estimates = true`

### 11.2 Policy linkage (ordering + paging + caps)

To interpret query behavior deterministically, the Runtime Pack Manifest MUST record the portable policy selections that govern:

* ordering (e.g., `policies.data_ordering`)
* paging mode (e.g., `policies.paging = CURSOR|OFFSET`)
* join caps and strictness (e.g., `policies.join1_cap.default` and `policies.join1_cap.mode = STRICT|TRUNCATE`)

(See `03-reproducibility/allowed-runtime-pack-policies.md`.)

---

## 12. Conformance tests (mandatory)

A v3 implementation MUST ship fixtures/tests that validate:

* deterministic ordering for a known pack
* stable paging (cursor or offset) across repeated runs
* cap breach behavior matches declared strictness mode
* error codes are stable and correct
* (if enabled) pagination profile behavior matches the profile requirements
* (if enabled) cardinality behavior matches documented semantics

---

## 13. Non-normative implementation notes

* Prefer cursor paging over offset for large packs (offset can be expensive).
* Enforce resource limits early to protect offline devices.
* If the pack declares signatures or integrity hashes, verify before answering queries; treat integrity failure as a hard error (fail-closed).

---

## 14. Open questions

* Do we require a single canonical response shape (triples vs bindings), or allow both with strong contract_id versioning?
* Should truthy projection be mandatory in runtime, or only mandatory in exports?
* Should join1 be standardized as a named operation input shape (beyond “capability”), or remain an implementation capability with a declared cap?