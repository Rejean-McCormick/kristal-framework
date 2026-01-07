Voici **18 / 18** — le contenu proposé pour **`docs/10-implementation-notes.md`**.

---

# 10. Implementation Notes and Reference Guidance (v0.1)

## 10.1 Purpose

This document provides **non-normative guidance** for implementers.

It explains:

* why certain choices were made,
* how to build a minimal reference implementation,
* what can be deferred without breaking the standard.

Nothing in this document is required for conformance, but it is intended to **lower the barrier to adoption**.

---

## 10.2 Minimal reference implementation (recommended)

A minimal, credible implementation can be built with **four components**:

1. **Claim-IR validator**

* JSON Schema validation
* Error/warning emission
* No resolution logic required

2. **Resolver (Sentient-like)**

* Surface → QID/PID lookup (offline subset)
* Candidate scoring
* Literal normalization (time, quantity)

3. **Kristal compiler**

* Claim-IR → Kristal Exchange
* Canonicalization + hashing
* Exchange → Runtime Pack

4. **Runtime executor**

* Load dictionaries
* Load indexes
* Execute Query Profile v0.1

Everything else is optional.

---

## 10.3 Language and tooling choices

Kristal is **language-agnostic**, but implementers may benefit from:

* **C++ / Rust** for runtime pack execution (memory-mapped IO, bitmaps)
* **Python** for:

  * prototyping
  * Claim-IR validation
  * batch compilation
* **DuckDB / Parquet** for triple table storage

A mixed-language stack is acceptable and expected.

---

## 10.4 JSON handling (exchange vs runtime)

**Important rule:**

> JSON is for interchange and validation, not for execution.

Implementation advice:

* Use streaming or on-demand parsing
* Avoid building full in-memory DOMs for large Kristals
* Canonicalize JSON once, then discard text representation

This follows best practices from high-performance parsing and data engineering.

---

## 10.5 Runtime pack implementation strategy

### Dictionaries

* Map external IDs (QID/PID) → dense integers
* Use fixed-width or varint encoding
* Keep dictionaries immutable per pack

### Triples

* Prefer columnar storage:

  * Parquet (easy)
  * custom binary columns (faster, more work)
* Sort triples by `(p,o,s)` if beneficial for index build

### Indexes

* Always build `(p,o) → {s}`
* Choose representation by cardinality:

  * large sets → compressed bitmaps
  * medium → delta-compressed sorted lists
  * tiny → arrays

This hybrid strategy is critical for performance predictability.

---

## 10.6 Query execution

Query engines should:

* evaluate smallest sets first
* short-circuit empty intersections
* cache intermediate results when safe

Avoid:

* recursive evaluation
* dynamic query planning
* unbounded joins

Kristal queries are meant to be **simple and fast**, not expressive.

---

## 10.7 Merging and deduplication

When combining multiple Runtime Packs:

* Use membership filters to skip non-overlapping packs
* Deduplicate by `(s,p,o)` or `claim_id`
* Prefer recomputation over in-place mutation

A common workflow:

1. Merge Exchange artifacts
2. Re-compile a fresh Runtime Pack

This preserves determinism and integrity.

---

## 10.8 Validation pragmatics

Not all Wikidata constraints need to be enforced.

Recommended:

* datatype compatibility
* item vs literal correctness
* time/quantity shape validity

Optional:

* single-value expectations
* class constraints

Kristal prioritizes **mechanical correctness** over social norms.

Constraint profiles should be:

* cached locally
* versioned
* explicitly referenced

---

## 10.9 Working with Wikidata and Wikibase

Kristal assumes:

* offline subsets of Wikidata data
* periodic refresh, not live querying

Export considerations:

* ensure all PIDs used are valid Wikibase properties
* unresolved content must not be exported as authoritative claims

Kristal Exchange → Wikibase JSON should be a **pure transformation**, not reinterpretation.

---

## 10.10 Performance expectations (order-of-magnitude)

On a modern laptop:

* parsing thousands of Claim-IR documents: seconds
* compiling small Kristals: milliseconds each
* querying Runtime Packs: sub-millisecond for most `has(p,o)` queries

Performance scales with:

* index selectivity
* cardinality distribution
* memory locality

Kristal intentionally avoids distributed execution.

---

## 10.11 Testing strategy

Recommended test layers:

1. Schema validation tests
2. Canonicalization + hashing tests
3. Exchange ↔ Runtime round-trip tests
4. Query correctness tests
5. Offline reproducibility tests

Golden-file testing is strongly encouraged.

---

## 10.12 What to defer safely (v0.1)

You can ship v0.1 **without**:

* signatures
* external evidence packs
* membership filters
* join support beyond simple AND
* multi-entity Kristals

None of these affect core correctness.

---

## 10.13 Relationship to existing ecosystems

Kristal is compatible with:

* Wikibase toolchains
* RDF export pipelines
* embedded analytics engines

It deliberately avoids tight coupling to:

* any single LLM provider
* any single database engine
* any centralized service

This keeps the ecosystem **future-proof**.

---

## 10.14 Final note to implementers

Kristal is best understood as:

> **a compiler target for knowledge**,
> not a database,
> not a document format,
> and not an AI model.

If your implementation preserves:

* determinism,
* Wikidata alignment,
* offline execution,

then it is likely conformant.

---

**End of document set.**
