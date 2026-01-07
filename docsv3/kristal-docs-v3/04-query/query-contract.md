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
* Deterministic result ordering.
* Deterministic paging (cursor or offset).
* Deterministic failure behavior for resource limits (join caps, page caps, timeouts).
* Optional profiles: TPF-like pagination, per-pattern cardinality metadata.

### 1.2 Out of scope (non-goals)

* Full SPARQL 1.1 semantics (OPTIONAL, UNION, FILTER with full expression semantics, property paths, subqueries, federated queries).
* Network dependencies: queries MUST be executable offline over a local Runtime Pack.
* Non-deterministic query behavior (randomization, adaptive sampling without declaring a profile).

---

## 2. Terminology

* **Term**: a subject/object IRI-like identifier (e.g., QID) or literal.
* **Triple**: `(s, p, o)` where `s` and `p` are identifiers; `o` is identifier or literal.
* **Triple pattern**: a triple with any of `s`, `p`, `o` possibly unbound.
* **Binding**: a concrete value for an unbound position.
* **Result set**: ordered list of matching triples or bindings.
* **Cursor**: opaque paging token that resumes iteration deterministically.
* **Offset paging**: deterministic paging by `(offset, limit)` against a deterministic ordering.
* **join1**: a constrained two-stage lookup pattern (see Section 6) with a defined cap.

Normative keywords: MUST, MUST NOT, SHOULD, SHOULD NOT, MAY.

---

## 3. Query model (mandatory)

### 3.1 Supported query primitive

An implementation MUST support querying a single triple pattern:

* Inputs:

  * `s` (bound or unbound)
  * `p` (bound or unbound)
  * `o` (bound or unbound)
  * `limit` (positive integer)
  * paging parameter(s) (cursor or offset mode, see Section 5)

* Output:

  * an ordered sequence of results, each result containing either:

    * matching triples `(s, p, o)`; or
    * bindings for unbound positions (`?s`, `?p`, `?o`) plus (optionally) the full triple

Implementations MUST document which output shape they use and MUST keep it stable within a `query_contract_version`.

### 3.2 Determinism

Given:

* identical Runtime Pack bytes,
* identical portable policy selections recorded in the pack manifest (ordering, etc.),
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

Runtime Packs MUST define (in their manifest) whether the runtime query surface:

* returns the full statement set, or
* returns a “truthy/best-rank” projection, or
* supports both as named projections.

If “truthy” is supported, the mapping rules MUST be deterministic and documented. If only exports define truthy behavior and runtime does not, the runtime MUST state that clearly.

---

## 5. Result ordering (mandatory)

### 5.1 Ordering source of truth

Result ordering MUST be derived from the Runtime Pack’s declared ordering policy (e.g., triple-table ordering such as SPO, POS, etc.). The ordering policy MUST be recorded in the Runtime Pack manifest.

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

* `page_limit_max` (maximum `limit` accepted),
* `join1_cap` (cap on intermediate expansion for join-like patterns, if supported),
* strictness behavior for cap breaches (see 6.3).

These MUST be surfaced in either:

* the Runtime Pack manifest, or
* the query wrapper configuration (if the wrapper is the enforcement point),
  but MUST remain deterministic for a given pack + configuration.

### 6.2 join1 (optional capability, but deterministic if present)

If the runtime supports a two-stage lookup (common for navigation), it MUST define join1 explicitly, for example:

1. Evaluate pattern A to produce intermediate bindings (bounded by `join1_cap`)
2. Use each binding to evaluate pattern B
3. Combine results deterministically

If join1 is not supported, the implementation MUST reject such requests deterministically.

### 6.3 Cap breach behavior (mandatory to define)

When a cap is exceeded, the implementation MUST choose one deterministic policy and document it:

* **ERROR**: return an error response with a stable error code (recommended for correctness)
* **TRUNCATE**: truncate intermediate results deterministically and return a “truncated” flag + counts

Silently truncating without a flag MUST NOT occur.

---

## 7. Paging contract (mandatory)

An implementation MUST support **either** cursor paging **or** offset paging. It MUST document which mode is supported and keep it stable per `query_contract_version`.

### 7.1 Cursor paging (recommended)

If cursor paging is supported:

* The cursor MUST be opaque to callers.
* The cursor MUST resume iteration deterministically for the same pack and query inputs.
* The cursor MUST become invalid if:

  * the pack changes, or
  * the query parameters change (other than `limit`), unless explicitly supported.

Required response fields:

* `next_cursor` (string or null)
* `has_more` (boolean)

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
* cap exceeded (if ERROR policy)
* cursor invalid/expired
* internal corruption / integrity failure (if pack verification fails)

### 8.2 Deterministic error behavior

Errors MUST be deterministic for a given:

* pack,
* query input,
* policy configuration.

---

## 9. Optional profile: cardinality metadata

If the **cardinality profile** is enabled, responses MAY include:

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

## 10. Optional profile: TPF-like wrapper (local/offline)

If the **TPF-like pagination profile** is enabled, a local wrapper MAY expose HTTP endpoints (still offline) such as:

* `GET /tpf?subject=...&predicate=...&object=...&limit=...&cursor=...`
* Response includes:

  * `results` (triples/bindings)
  * `next_cursor`
  * `cardinality` (optional, if enabled)
  * `policy` summary (optional echo)

This profile MUST NOT require network access. “HTTP” here means a local loopback or embedded server.

---

## 11. Manifest requirements (mandatory)

The Runtime Pack manifest MUST include, at minimum, the fields necessary to interpret query behavior deterministically, such as:

* `query_contract_version`
* `ordering_policy` (and any tie-breakers if not implied)
* `paging_mode` (`"cursor"` or `"offset"`)
* `page_limit_max`
* `join1_cap` and `join1_cap_policy` (`"ERROR"` or `"TRUNCATE"`)
* projection support (full vs truthy) or explicit statement that truthy is export-only

---

## 12. Conformance tests (mandatory)

A v3 implementation MUST ship fixtures/tests that validate:

* deterministic ordering for a known pack
* stable paging (cursor or offset) across repeated runs
* cap breach behavior matches declared policy
* error codes are stable and correct
* (if enabled) cardinality profile behavior matches documented semantics

---

## 13. Non-normative implementation notes

* Prefer cursor paging over offset for large packs (offset can be expensive).
* Enforce resource limits early to prevent denial-of-service on constrained devices.
* If the pack is signed, verify signatures before answering any queries; treat integrity failure as a hard error.

---

## 14. Open questions

* Do we require a single canonical response shape (triples vs bindings), or allow both with strong versioning?
* Should truthy projection be mandatory in runtime, or only mandatory in exports?
* Should join1 be standardized as a named operation, or remain an implementation capability with a declared cap?
