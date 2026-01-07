Voici **3 / 18** — le contenu proposé pour **`docs/1-standard.md`**.

---

# 1. Standard Positioning, Scope, and Compatibility

## 1.1 Purpose of this document

This document defines **what Kristal standardizes**, **what it explicitly does not**, and **how it aligns** with existing standards and communities.

It is written to be readable by:

* Wikidata / Wikibase contributors
* W3C community groups
* Knowledge graph and data engineering practitioners
* AI system designers concerned with grounding and determinism

---

## 1.2 Positioning statement

Kristal is a **format specification**, not a platform.

It defines:

* how **encyclopedic knowledge blocks** are represented,
* how they can be **compiled, validated, and executed offline**,
* how they remain **fully interoperable with Wikidata semantics**.

Kristal does **not** define:

* a hosting infrastructure,
* a centralized service,
* a replacement for Wikidata or Wikibase.

Kristal is intended to be **adoptable incrementally**, alongside existing systems.

---

## 1.3 Relationship to existing standards and initiatives

### Wikidata

Kristal is **structurally aligned** with Wikidata.

* QIDs and PIDs are reused verbatim.
* Claim semantics (statement, qualifier, reference) are preserved.
* Kristal content can be exported to Wikidata-compatible JSON and RDF.

Kristal does not introduce a competing ontology or identifier space.

---

### Wikibase

Kristal is **compatible with the data model** of Wikibase.

* Kristal Exchange mirrors Wikibase’s statement structure.
* The mapping from Kristal Exchange → Wikibase JSON is lossless for supported features.
* Kristal may be seen as a **portable Wikibase-compatible artifact**, not a live Wikibase instance.

---

### W3C standards

Kristal aligns with W3C technologies where appropriate:

* **JSON-LD** for Linked Data interoperability
* **RDF** as an export and interchange target
* **Shape Expressions (ShEx)** concepts for validation
* **Linked Data Fragments (TPF)** ideas for constrained querying

Kristal does **not** propose a new RDF serialization or query language.

Its contribution is architectural, not syntactic.

Kristal is suitable for discussion within W3C Community Groups rather than as a replacement standard.

---

## 1.4 Scope (what Kristal standardizes)

Kristal standardizes the following artifacts:

1. **Claim-IR**
   A strict, schema-defined intermediate representation produced by extractors or LLMs.

2. **Kristal Exchange format**
   A Wikibase-aligned JSON format for:

   * claims
   * qualifiers
   * references
   * provenance

3. **Kristal Runtime Pack**
   A compact, offline-executable representation including:

   * dictionaries (IDs → integers)
   * triple tables
   * indexes
   * filters
   * minimal query API

4. **Validation and integrity rules**

   * structural validation
   * semantic constraints
   * hashing and identity

5. **A constrained query profile**

   * triple-pattern–based
   * set-oriented
   * explicitly *not* full SPARQL

---

## 1.5 Non-goals (explicit exclusions)

Kristal does **not** aim to:

* Implement full SPARQL semantics
* Replace Wikidata or Wikibase
* Define a reasoning or inference engine
* Act as a real-time collaborative editing system
* Solve ontology design or governance at large

These exclusions are deliberate and necessary for:

* offline execution
* determinism
* predictable performance
* long-term maintainability

---

## 1.6 Compatibility and degradation guarantees

Kristal enforces **backward compatibility by design**.

### Export guarantees

Every Kristal Exchange document:

* can be exported to Wikibase JSON,
* can be expressed as RDF / JSON-LD,
* preserves original QIDs, PIDs, and references.

### Degradation rules

When information is incomplete:

* unresolved QIDs/PIDs are preserved as surfaces + candidates,
* claims may be marked partial rather than rejected,
* no information is silently dropped.

This ensures Kristal artifacts remain usable even when not fully resolved.

---

## 1.7 Versioning and evolution

Kristal versions follow these principles:

* **Schema versioning is explicit** (`v0.1`, `v0.2`, …).
* Minor versions may add optional fields.
* Major versions may change semantics, but:

  * export compatibility must be maintained,
  * older Kristals must remain readable.

Evolution is constrained by **interoperability first**, not feature velocity.

---

## 1.8 Intended standardization path

Kristal is suitable for:

* technical review
* prototyping
* community feedback
* possible submission to a W3C Community Group
* alignment discussions with Wikimedia-affiliated projects

Kristal is not proposed as a W3C Recommendation at this stage.

---

## 1.9 Summary

Kristal occupies a precise niche:

> **Executable, offline-first, Wikidata-aligned knowledge blocks**

By clearly limiting its scope and aligning with existing standards rather than competing with them, Kristal aims to be:

* technically credible,
* politically adoptable,
* and practically useful.

---

**Next document:** `2-prior-art.md`
