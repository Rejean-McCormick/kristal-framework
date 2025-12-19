
# Reload a Kristal (Local Query + AI Runtime)

## Status
DRAFT

## 1. Goal

This tutorial explains how to reload a Kristal into a local environment after you:
- downloaded it,
- edited it offline,
- or received an updated version.

“Reload” means making the Kristal available to:
- a local query engine (SPARQL-like),
- an entity/predicate resolver (indexes),
- and optionally an AI runtime (agent/LLM grounding).

---

## 2. What Must Be Loaded

A runtime typically loads:

1) **Kristal core**
- `*.kristal.json` containing:
  - manifest
  - qitems
  - claims
  - charters (embedded or referenced)
  - optional index

2) **Charters** (if referenced)
- property catalogs needed to interpret P/R/S

3) **Indexes** (optional)
- label/alias lookup (entity resolution)
- property lexicon (predicate resolution)
- optional embeddings

4) **Trace bundle** (optional)
- only needed if you want audit/explanations

---

## 3. Reload Modes

### 3.1 Minimal reload (query only)
Load:
- `qitems[]`
- `claims[]`

This enables:
- triple pattern queries
- slicing

Example (conceptual CLI):
```bash
kristal load --in plants.core.local.kristal.json
````

### 3.2 Full reload (query + resolution)

Load:

* Kristal core + `index`
* charters (embedded/ref)
* local resolver caches

Example:

```bash
kristal load --in plants.core.local.kristal.json --with-index --with-charters
```

### 3.3 Audit reload (include traces)

Load:

* Kristal core
* trace bundle(s)

Example:

```bash
kristal load --in plants.core.local.kristal.json --with-traces ./traces/trace.bundle.local.json
```

---

## 4. Local Store Layout (Recommended)

```
~/.kristals/
  store/
    plants.core.local.kristal.json
    chemistry.core.kristal.json
  charters/
    general.charter.json
    plants.domain.charter.json
  indexes/
    plants.core.index.json
  traces/
    trace.bundle.local.json
  cache/
    qid_candidates.cache
    property_candidates.cache
```

---

## 5. Integrity Checks on Reload

A loader SHOULD verify:

### 5.1 Schema validation

* verify the Kristal JSON structure
* verify required manifest fields exist

### 5.2 Referential integrity

* every claim `subject` must be present in `qitems[]` (or explicitly allowed as external)
* every claim QID object referenced should be present (or marked external)
* every `property` used must exist in loaded charters (P/R/S)

### 5.3 Hash consistency

* recompute `kristal_id` and compare (if `kristal_id` is content-addressed)
* optionally verify signature

### 5.4 Index consistency (optional)

If `index` exists:

* confirm index entries resolve to known `qitems`
* optionally rebuild index and compare (to avoid corrupted indexes)

---

## 6. Building Runtime Structures

On reload, the runtime SHOULD build:

### 6.1 Triple store (in-memory)

Convert each claim into:

* `(subject, property, object)` triples
* attach metadata:

  * confidence, status, context, trace refs

### 6.2 Inverted indexes (fast lookup)

* by subject: outgoing edges
* by object: incoming edges
* by property: claims per property

This makes local querying fast without a full RDF store.

### 6.3 Resolver indexes

If enabled:

* label/alias → QID candidates
* property label/alias → P/R/S candidates

---

## 7. Reload into an AI Runtime (Grounding)

### 7.1 Retrieval flow (typical)

1. user asks a question
2. system extracts key mentions (entities)
3. resolve mentions to QIDs using indexes
4. run local query to retrieve a relevant subgraph
5. format the subgraph as AI context
6. generate answer with citations (trace refs if needed)

### 7.2 Context policy (recommended)

Use contexts/status/confidence to filter:

* default: only `asserted` claims above threshold
* optionally include `inferred` for brainstorming
* include `contested` only when asked

---

## 8. After Reload: Typical Commands

Examples (conceptual):

```bash
kristal query --in plants.core.local.kristal.json --sparql 'SELECT ?o WHERE { Q123 P279 ?o . }'
kristal slice --in plants.core.local.kristal.json --seed Q123 --closure outgoing --depth 2 --out Q123.neighborhood.kristal.json
kristal rebuild-index --in plants.core.local.kristal.json --out ./indexes/plants.core.index.json
```

---

## 9. Troubleshooting

### Problem: Unknown property IDs

Cause:

* missing charter file (refs not loaded)

Fix:

* fetch or embed the charter
* reload with `--with-charters`

### Problem: Missing Q-items for referenced objects

Cause:

* claims reference QIDs not present in `qitems[]`

Fix:

* allow external references, or
* expand slice closure depth and re-export a larger pack

### Problem: IDs changed after edits

Cause:

* claim payload changes require re-hashing

Fix:

* recompute `claim_id` for changed claims
* ensure canonicalization rules are stable

---

## 10. Next Steps

* Propose your edits for merging: `docs/en/contributors/propose_change.md`
* Test a Kristal before publishing: `docs/en/contributors/test_kristal.md`

