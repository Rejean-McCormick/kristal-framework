Voici **14 / 18** — le contenu proposé pour **`docs/6-query-profile.md`**.

---

# 6. Query Profile (v0.1) — Offline Queries without Full SPARQL

## 6.1 Purpose

This document defines the **query surface** supported by Kristal Runtime Packs.

Goal:

* enable practical “Wikidata-like” lookups offline
* with predictable performance and minimal implementation burden

Non-goal:

* full SPARQL generality

Kristal adopts a **profile**: a constrained subset of graph queries that maps efficiently to set operations.

---

## 6.2 Principles

1. **Triple-pattern first**

   * `(p,o) → {s}` is the primary primitive
2. **Set algebra**

   * AND/OR/NOT over subject sets
3. **Limited joins**

   * one-hop joins only (v0.1)
4. **No unbounded search**

   * no property paths
   * no recursion
5. **Deterministic execution**

   * queries must be evaluable using runtime pack indexes

---

## 6.3 Core query model (subject-set semantics)

A query evaluates to a **set of subject IDs**.

Let:

* `S(p,o)` be the set of subjects having `(p,o)`.

Then:

* `has(p,o)` returns `S(p,o)`
* `and(q1,q2)` returns `q1 ∩ q2`
* `or(q1,q2)` returns `q1 ∪ q2`
* `not(q)` returns `U \ q` (where `U` is the universe of subjects in the pack)

---

## 6.4 Query JSON DSL (normative)

Queries are encoded as JSON objects.

### 6.4.1 `has`

```json
{ "op": "has", "p": "P31", "o": "Q5" }
```

Semantics:

* return subjects with property `P31` and object `Q5`

### 6.4.2 `and`

```json
{ "op": "and", "args": [ { ... }, { ... } ] }
```

### 6.4.3 `or`

```json
{ "op": "or", "args": [ { ... }, { ... } ] }
```

### 6.4.4 `not`

```json
{ "op": "not", "arg": { ... } }
```

### 6.4.5 `count`

```json
{ "op": "count", "arg": { ... } }
```

Returns integer cardinality.

### 6.4.6 `topk` (optional)

```json
{ "op": "topk", "arg": { ... }, "k": 20 }
```

Meaning:

* return up to `k` subjects from the result set
* ordering is implementation-defined unless an explicit rank index exists

---

## 6.5 Identifiers and resolution

Inputs may be given as:

* QIDs / PIDs (`"P31"`, `"Q5"`)
* or runtime integers (`p_id`, `o_id`) for low-level API calls

Normative rule:

* if QID/PID is provided, the runtime must resolve it via dictionaries
* if not found, the query returns empty set

---

## 6.6 One-hop joins (v0.1)

Many useful Wikidata-like queries require one join.

Example:
“Find people (P31=human) whose country of citizenship (P27) is France (Q142).”

No join needed:

* `and(has(P31,Q5), has(P27,Q142))`

But some patterns do:

Example:
“Find people whose employer is an organization located in France.”

This is a two-step pattern:

1. organizations located in France: `Org = has(P17, Q142)` (simplified)
2. people with employer in Org: join on object set

Kristal v0.1 supports one-hop join via:

```json
{
  "op": "join1",
  "p_out": "P108",
  "inner": { "op": "has", "p": "P17", "o": "Q142" }
}
```

Semantics:

* evaluate `inner` to a set `T` (as subject set)
* treat `T` as a set of QIDs (entity IDs)
* return subjects `s` such that `(s, p_out, o)` where `o ∈ T`

This requires an index form:

* `(p, o) → {s}` for all candidate `o`

Execution is typically:

* iterate over `o ∈ T`
* union `S(p_out, o)` across those `o`
* optionally intersect with other filters

**Constraint:** The runtime may cap `|T|` (inner result size) for predictability.

---

## 6.7 Planning rules (recommended)

To ensure predictable speed, implementations should:

* execute smallest intermediate sets first
* use `(p,o)` cardinality stats to reorder AND arguments
* short-circuit empty intersections early

This is directly enabled by:

* `statistics.po_cardinality`

---

## 6.8 Relationship to SPARQL

This query profile corresponds to a subset of SPARQL patterns:

* basic graph patterns with bounded complexity
* boolean combinations
* limited joins

Kristal intentionally excludes:

* OPTIONAL
* property paths
* FILTER over arbitrary expressions
* aggregation beyond count/topk

Kristal runtime packs may still be exposed behind a SPARQL adapter, but that adapter is outside the v0.1 standard.

---

## 6.9 Summary

Kristal Query Profile v0.1 provides:

* Wikidata-like offline retrieval
* set-based determinism and speed
* a small, implementable surface area

This is sufficient to power:

* offline lookup
* faceted exploration
* AI grounding checks
* deterministic generation pipelines

---

**Next document:** `7-validation.md`
