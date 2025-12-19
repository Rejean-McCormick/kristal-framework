
# Querying & Slicing Kristals (SPARQL-like Queries)

## Status
DRAFT

## 1. Purpose

Kristals are designed to be **queryable like a database** (offline or online) and to support **slicing**: extracting a coherent subset of a Kristal (or multiple Kristals) as a new Kristal.

This document specifies:
- a recommended query model (SPARQL/RDF-compatible),
- a practical “SPARQL-like” subset of queries for Kristal JSON,
- slicing rules (closure, dependencies, trace inclusion),
- domain packaging (e.g., UNESCO-aligned fields of expertise).

This spec does not mandate a specific engine. Implementations may use:
- an embedded RDF triple store + SPARQL endpoint, **or**
- a native JSON claim engine that supports an equivalent query language.

---

## 2. Data Model for Queries

Kristal core is stored as:
- `qitems[]` (entities)
- `claims[]` (flat list of statements)
- optional `index` (labels/aliases)

For querying, the core model is treated as triples:

- `(subject, property, object)` where:
  - `subject` = `claim.subject` (QID)
  - `property` = `claim.property` (P/R/S)
  - `object` = `claim.value` (QID or literal)

Qualifiers and references are treated as optional statement metadata.

---

## 3. SPARQL Compatibility (Recommended)

If the implementation exposes a SPARQL endpoint, it SHOULD provide an RDF mapping:
- `Q...` → `wd:Q...`
- `P...` → `wdt:P...` (or custom namespace for R/S)
- `R...` → `orgoR:R...`
- `S...` → `orgoS:S...`

Example namespace suggestion:
- `wd:` for Wikidata-aligned resources
- `orgoR:` and `orgoS:` for Orgo properties

> Exact RDF mapping details are finalized in the Linked Data export spec.

---

## 4. SPARQL-like Query Subset (Kristal JSON)

Implementations that do not ship RDF stores MUST support at minimum:

### 4.1 Basic pattern match
- Find subjects with a given property/value

**Pseudo-SPARQL**
```sparql
SELECT ?s WHERE {
  ?s P31 Q756 .
}
````

**JSON semantics**

* Return all claims where:

  * `property == "P31"`
  * `value.type == "wikibase-entityid"`
  * `value.content == "Q756"`
* Output distinct `subject` values.

### 4.2 Retrieve outgoing edges

```sparql
SELECT ?p ?o WHERE { Q123 ?p ?o . }
```

### 4.3 Retrieve incoming edges

```sparql
SELECT ?s ?p WHERE { ?s ?p Q123 . }
```

### 4.4 Filter by confidence and status

```sparql
SELECT ?s ?p ?o WHERE {
  ?s ?p ?o .
  FILTER(?confidence >= 0.9)
  FILTER(?status = "asserted")
}
```

### 4.5 Join patterns (two-hop)

```sparql
SELECT ?x WHERE {
  Q123 P361 ?m .
  ?m P279 ?x .
}
```

---

## 5. Slicing (Extracting a Part of a Kristal)

Slicing produces a **new Kristal** by applying a query + closure rules.

### 5.1 Slice Request Object

```json
{
  "slice_id": "slice://sha256:...",
  "query": {
    "type": "sparql",
    "text": "SELECT ?s WHERE { ?s P31 Q756 . }"
  },
  "options": {
    "closure": "none | outgoing | bidirectional",
    "depth": 0,
    "include_labels": true,
    "include_aliases": true,
    "include_references": "none | stubs | full",
    "include_traces": "none | refs | bundle",
    "min_confidence": 0.75,
    "statuses": ["asserted", "inferred"]
  }
}
```

### 5.2 Closure Rules

A slice MUST define how to include dependencies:

* `closure: none`

  * only include claims that match the query patterns

* `closure: outgoing`

  * include matched subjects, and include claims where:

    * `subject in selected_entities`
  * optionally traverse `depth` hops

* `closure: bidirectional`

  * include outgoing + incoming claims for selected entities
  * useful for “full local neighborhood” packs

**Depth**

* `depth = 0`: no traversal beyond direct matches
* `depth = 1`: include 1-hop neighbors
* `depth = n`: expand n hops (bounded)

### 5.3 Entity Set Construction

A slice always computes an entity set:

* all subjects included
* all QID objects referenced by included claims
* optionally include “type parents” (`P279` chains) if configured

### 5.4 Result: Slice Output

The output MUST be a valid Kristal file (SPEC format):

* `qitems[]` limited to selected entities (plus referenced objects)
* `claims[]` limited to included claims
* manifest includes lineage metadata

Example lineage fields:

```json
{
  "derived_from": [
    { "kristal_id": "kristal://sha256:...", "slice_id": "slice://sha256:..." }
  ]
}
```

---

## 6. Domain Packs (Ready-to-go Kristals)

Kristals can be distributed as curated domain packs:

* “Plants (Core)”
* “Organic Chemistry (Core)”
* “Canadian History (Core)”

### 6.1 UNESCO-Aligned Domains (Recommended)

Each Kristal SHOULD include at least one domain tag:

* `domains[] = [{ scheme: "unesco", code: "<field-code>" }]`

This enables:

* discovery by expertise field,
* standardized pack naming,
* stable slicing rules.

### 6.2 Pack Construction Strategies

Two common strategies:

1. **Curated pack**

* humans define the included set (hand-picked QIDs, properties)
* validated and versioned

2. **Query-built pack**

* the registry runs a canonical query against a large knowledge base
* produces deterministic slices (e.g., all entities where `field of study = botany`)

---

## 7. Interfaces

### 7.1 CLI (illustrative)

```bash
kristal query --in plants.kristal.json --sparql "SELECT ?s WHERE { ?s P31 Q756 . }"
kristal slice --in plants.kristal.json --sparql "..." --closure outgoing --depth 1 --out plants.slice.kristal.json
```

### 7.2 Registry API (illustrative)

* `POST /slice`
* `GET /kristals?domain=unesco:<code>`
* `GET /kristals/<id>/download`
* `GET /kristals/<id>/slice?sparql=...`

---

## 8. Conformance (Minimum Query Engine)

A conforming Kristal query engine MUST support:

* triple pattern matching over `(subject, property, value)`
* QID object matching (`wikibase-entityid`)
* filters on `confidence` and `status`
* slice output generation with closure rules
* deterministic ordering of outputs for reproducible slices

