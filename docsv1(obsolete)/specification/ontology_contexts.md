
# Ontology & Contexts (Kristals)

## Status
DRAFT

## 1. Purpose

Kristals are structured knowledge artifacts aligned with Wikidata/Wikibase, extended via Orgo’s P/R/S model, and designed for offline-first, mergeable knowledge work.

This document defines:
- how Kristals represent **ontology structure** (types, hierarchies, domains),
- how Kristals represent **contexts** (layers of meaning, validity, trust, and interpretation),
- how contexts interact with querying, validation, merging, and AI reasoning.

This document does not define:
- trace log payload details (see Trace Log spec),
- full RDF export semantics (see Linked Data spec).

---

## 2. Ontology Concepts in Kristals

### 2.1 Entities and typing
Entities are `Q...` items. Ontological typing is expressed through claims like:

- `P31` (instance of)
- `P279` (subclass of)

Example:
- `Q<oak> P31 Q<tree>`
- `Q<tree> P279 Q<plant>`

Kristals may store only a subset of ontology chains (domain-scoped).

### 2.2 Property vocabulary
Properties are:
- `P...` (Wikidata-aligned)
- `R...` (Orgo refinements of P)
- `S...` (Orgo-specific properties beyond Wikidata)

Property definitions and constraints are supplied by Orgo charters (embedded or referenced).

### 2.3 Domain modeling
Kristals SHOULD annotate their scope via manifest `domains[]`, such as:
- UNESCO field codes (expertise taxonomy)
- Wikidata QIDs for fields of study / disciplines

Domain modeling is used for:
- discovery in registries,
- default slicing behavior,
- selecting validation policies.

---

## 3. What is a Context?

A **context** is a named layer that determines how a set of claims should be interpreted, filtered, trusted, or applied.

Contexts are necessary because:
- claims can be **asserted**, **inferred**, **contested**, or **deprecated**,
- offline edits create local variants,
- different expert groups may maintain parallel interpretations,
- AI pipelines need explicit filters to prevent mixing trust levels.

Contexts are not the same as trace logs:
- trace logs explain *how* a claim was produced,
- contexts define *how it should be treated*.

---

## 4. Context Model

### 4.1 Context identifiers
Contexts are referenced by IDs:

- `ctx:core` (default)
- `ctx:inferred`
- `ctx:contested`
- `ctx:local`
- `ctx:deprecated`

Implementations MAY use URI forms:
- `context://core`
- `context://<name>`
- `context://sha256:<hash>` for strict content-addressing

### 4.2 Context definition object
A Kristal MAY define context objects in the manifest:

```json
{
  "contexts": [
    {
      "id": "ctx:core",
      "label": "Core asserted knowledge",
      "policy": {
        "min_confidence": 0.80,
        "allowed_statuses": ["asserted"],
        "require_evidence": true
      }
    },
    {
      "id": "ctx:inferred",
      "label": "Inferred knowledge",
      "policy": {
        "min_confidence": 0.60,
        "allowed_statuses": ["inferred"],
        "require_evidence": false
      }
    }
  ]
}
````

If no contexts are defined:

* `ctx:core` is implied.

---

## 5. Context Assignment to Claims

### 5.1 Claim-level context field (recommended)

Claims MAY include an explicit context:

```json
{
  "claim_id": "claim://sha256:...",
  "subject": "Q123",
  "property": "P31",
  "value": { "type": "wikibase-entityid", "content": "Q33742" },
  "context": "ctx:core",
  "confidence": 0.93,
  "status": "asserted",
  "trace_refs": []
}
```

If `context` is omitted:

* default context is `ctx:core`.

### 5.2 Context derived from status (fallback)

If claim context is not present:

* implementations MAY map contexts from `status`:

  * `asserted` → `ctx:core`
  * `inferred` → `ctx:inferred`
  * `conflicted` → `ctx:contested`
  * `rejected` → `ctx:deprecated`

---

## 6. Context Interaction with Querying

Queries SHOULD allow context filters:

Example SPARQL-like:

```sparql
SELECT ?s ?p ?o WHERE {
  ?s ?p ?o .
  FILTER(context = "ctx:core")
  FILTER(confidence >= 0.85)
}
```

Slicing MUST specify context selection:

* include only `ctx:core`
* include `ctx:core + ctx:inferred`
* include all contexts for audit packs

---

## 7. Context Interaction with Validation

Validation policies may be context-specific:

* `ctx:core` may require evidence
* `ctx:inferred` may allow missing references but requires explicit inference marker
* `ctx:local` may allow placeholders

Validation output MAY move claims between contexts:

* a claim failing evidence checks may be demoted from `ctx:core` to `ctx:contested`
* a manually verified claim may be promoted to `ctx:core`

---

## 8. Context Interaction with Merging

Merging must treat context as a first-class dimension:

### 8.1 Deduplication

Claims with the same `(subject, property, value)` but different contexts are NOT duplicates:

* they represent different interpretation layers.

### 8.2 Conflict handling

Conflicts may be represented as:

* competing claims in `ctx:contested`,
* a merged “core” claim plus contested alternatives.

### 8.3 Local vs upstream contexts

When a user edits offline:

* new or modified claims default to `ctx:local`
* upstream merge can:

  * accept into `ctx:core`,
  * reject into `ctx:deprecated`,
  * keep in `ctx:contested`.

---

## 9. Context Interaction with AI Reasoning

AI systems consuming Kristals MUST select contexts intentionally:

Recommended defaults:

* use only `ctx:core` for factual answering,
* include `ctx:inferred` for hypothesis generation,
* include `ctx:contested` only when explicitly requested (e.g., “show disagreements”).

AI agents SHOULD also:

* surface context in explanations,
* avoid mixing contexts silently.

---

## 10. Ontology Contexts (Optional Advanced)

A Kristal MAY define context-specific ontology layers:

* different subclass chains (taxonomy versions),
* alternative classifications (e.g., APG III vs APG IV),
* local naming conventions.

This is handled by assigning ontology claims (`P279`, `P31`) to contexts.

---

## 11. Conformance

A conforming implementation MUST:

* support a default context (`ctx:core`),
* allow context assignment per claim (explicit or derived),
* allow context filtering during query and slicing,
* preserve contexts during merge and export.


