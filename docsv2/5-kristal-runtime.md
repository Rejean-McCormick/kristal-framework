Voici **11 / 18** — le contenu proposé pour **`docs/5-kristal-runtime.md`**.

---

# 5. Kristal Runtime Pack (v0.1)

## 5.1 Purpose

The **Kristal Runtime Pack** is the **offline-executable form** of a Kristal.

It is designed for:

* predictable performance,
* deterministic results,
* execution without SPARQL, network access, or LLMs.

It is **compiled from Kristal Exchange** and is **not** intended for human editing.

---

## 5.2 Design principles

1. **Offline-first**

   * No network dependency
   * No SPARQL endpoint
   * No dynamic schema

2. **Separation of concerns**

   * Exchange format = interoperability
   * Runtime pack = execution

3. **Integer-first execution**

   * All QIDs/PIDs/values mapped to dense integers
   * Strings kept out of the hot path

4. **Set algebra over graph traversal**

   * Queries expressed as intersections/unions/differences
   * One-hop joins only (v0.1)

5. **Predictable cost model**

   * No unbounded joins
   * No recursive graph walks

---

## 5.3 Conceptual model

At runtime, a Kristal is executed as:

```
IDs (integers)
  ↓
Triples table (S,P,O)
  ↓
Indexes by pattern (P,O) → S-set
  ↓
Set operations (AND / OR / NOT)
```

This model is inspired by compressed RDF and bitmap-index systems, and aligns with performance-oriented research popularized by work from **Daniel Lemire**.

---

## 5.4 Runtime Pack contents

A Runtime Pack is a directory (or archive) with the following logical components:

### 5.4.1 Manifest

`manifest.json`

* runtime version
* source Kristal ID(s)
* compilation timestamp
* build parameters
* index summary (counts, cardinalities)

The manifest allows tooling to:

* verify compatibility
* inspect coverage
* plan queries

---

### 5.4.2 Dictionaries

Compact mappings from identifiers to integers:

* `qid.dict` — QID → int
* `pid.dict` — PID → int
* `value.dict` — literal values → int (strings, quantities, times)
* optional: `rid.dict`, `sid.dict` (for Orgo-style extensions)

Dictionaries are immutable within a pack.

---

### 5.4.3 Triple table

A columnar representation of triples:

| Column | Type |
| ------ | ---- |
| `s`    | int  |
| `p`    | int  |
| `o`    | int  |

Allowed encodings:

* Parquet (recommended)
* Binary columnar formats
* Memory-mapped arrays

The triple table contains **only resolved statements**.

Unresolved or partial content remains in Kristal Exchange, not runtime.

---

### 5.4.4 Indexes

Indexes are precomputed for **frequent access patterns**.

Mandatory (v0.1):

* **(p,o) → {s}** subject sets

Optional:

* **(p) → {s}**
* **(p,s) → {o}** (low cardinality only)

Index storage:

* compressed bitmaps for large sets
* delta-compressed sorted lists for small sets
* raw arrays for tiny sets

The choice is based on **cardinality thresholds** recorded in the manifest.

---

### 5.4.5 Membership filters (optional but recommended)

Each pack may include a probabilistic filter over:

* `claim_id`s or `(s,p,o)` hashes

Purpose:

* fast “may-contain?” checks
* efficient offline merge/dedup workflows
* skip I/O when merging multiple packs

False positives are acceptable; false negatives are not.

---

### 5.4.6 Statistics

Basic statistics for query planning:

* cardinality per `(p,o)`
* cardinality per `p`
* total triples

Statistics are read-only and computed at compile time.

---

## 5.5 Query execution model

### 5.5.1 Supported query primitives (v0.1)

* `has(p, o)` → set of subjects
* `and(q1, q2)`
* `or(q1, q2)`
* `not(q)`
* `count(q)`
* `topk(q, k)` (optional)

Joins:

* limited to **one hop**
* executed as set intersections

This constrained profile is sufficient for:

* encyclopedic lookup
* faceted browsing
* AI grounding checks

---

### 5.5.2 Explicit non-support

The Runtime Pack does **not** support:

* full SPARQL semantics
* OPTIONAL, UNION with arbitrary depth
* property paths
* inference or reasoning rules

These are non-goals by design.

---

## 5.6 Compilation rules (from Exchange → Runtime)

During compilation:

1. Resolve QIDs/PIDs to integers
2. Drop unresolved statements (recorded separately)
3. Normalize values (time, quantity, text)
4. Emit triples
5. Build indexes by cardinality
6. Compute statistics and filters
7. Write manifest

Compilation is deterministic: same input → same runtime pack.

---

## 5.7 Backward compatibility

* Runtime packs are **versioned**
* Tooling must refuse incompatible versions
* Exchange artifacts remain the canonical source of truth

Runtime packs are **derivable artifacts** and may be regenerated.

---

## 5.8 Relationship to Wikidata execution

Kristal Runtime Packs:

* do **not** replace Wikidata Query Service
* do **not** attempt to replicate global query semantics

They instead provide:

> a **portable, local execution substrate** for Wikidata-aligned knowledge.

This makes them suitable for:

* offline devices
* regulated environments
* reproducible AI pipelines

---

## 5.9 Summary

The Kristal Runtime Pack is:

* compact
* deterministic
* offline
* optimized for set-based queries

It is the key that turns structured knowledge into **executable knowledge**.

---

**Next document:** `5-kristal-runtime.schema.json`
