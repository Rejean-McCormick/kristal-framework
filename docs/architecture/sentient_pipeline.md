
# SenTient Pipeline (AI → Kristal Compilation)

## Status
DRAFT

## 1. Purpose

This document describes how **SenTient** (OpenTapioca + Falcon stack) fits into the Kristal framework as the primary resolution and enrichment layer.

The pipeline turns:
- human text, documents, or LLM outputs  
into:
- resolved Q/P/R/S identifiers, typed values, validated claims, and packaged Kristals.

This document focuses on the **processing flow** and **interfaces**. The Kristal core format is defined in `specification/format_kristal.md`.

---

## 2. SenTient’s Role

SenTient is used for:
- **Entity Linking (Q resolution)**: text → QIDs
- **Predicate/Relation Linking (P/R/S resolution)**: predicate intent → property IDs (P/R/S)
- **Typing & normalization**: dates, quantities, coords, URLs, language tags
- **Candidate ranking**: returning top candidates with confidence

SenTient is also used as an **ACL layer** between external representations and internal DTOs.

---

## 3. Supported Inputs

### 3.1 Text-first ingestion (documents / prompts)
Inputs:
- raw text (question, paragraph, article)
- optional metadata (language, domain hints, source URL)

Outputs:
- mentions + candidates (entities/properties)
- evidence locators
- structured IR claims (optional)

### 3.2 IR-first ingestion (LLM structured output)
Inputs:
- Claim IR (subject/predicate/object + evidence)
Outputs:
- resolved IDs + typed claims

### 3.3 Graph-first ingestion (WikidataGraph)
Inputs:
- `WikidataGraph` containing entities and claims
Outputs:
- verified/resolved claims (P/R/S) and Q checks

A WikidataGraph-like structure includes entities and their claims with typed values and confidence fields.

---

## 4. Pipeline Overview

### Stage A — Trace capture (detached logs)
- capture prompt, raw model output, source excerpts/hashes
- emit `trace_root` reference
- do not embed full logs into Kristal core

### Stage B — Mention extraction
From text or IR:
- identify entity mentions (candidate subjects/objects)
- identify predicate phrases (candidate properties)

### Stage C — Candidate generation
**Entities**
- lexical lookup in Kristal indexes (labels/aliases)
- optional external lookup (registry caches)
- generate candidate QIDs

**Properties**
- lookup against Orgo charters:
  - P definitions
  - R refinements
  - S properties
- return candidate P/R/S IDs

### Stage D — Disambiguation and ranking
Use:
- local context
- evidence snippets
- co-mentions and graph neighborhood

Outputs:
- selected QID and selected property ID
- candidate lists + decision metadata (traceable)

### Stage E — Value typing
Convert objects to:
- entity ids (QID)
- literal types (time/quantity/url/string/etc.)

Value typing may be constrained by charter-defined `value_type`.

### Stage F — Claim construction
Produce canonical Kristal claim records:
- subject + property + typed value
- qualifiers + reference stubs (optional)
- confidence and status
- trace references

### Stage G — Validation handoff
Claims are passed to validators/scrutinizers:
- schema checks
- constraint checks
- evidence policy enforcement
- duplication/conflict detection

### Stage H — Packaging
Emit:
- `kristal.json`
- optional `trace.bundle.json`
- optional indexes

---

## 5. Interfaces

### 5.1 Entity linking interface (conceptual)

Input:
```json
{
  "text": "photosynthesis",
  "lang": "en",
  "context": "plants biological process"
}
````

Output:

```json
{
  "candidates": [
    { "id": "Q123", "score": 0.92 },
    { "id": "Q999", "score": 0.31 }
  ],
  "selected": "Q123",
  "selection_confidence": 0.92
}
```

### 5.2 Predicate linking interface (P/R/S)

Input:

```json
{
  "predicate_text": "assigned to",
  "lang": "en",
  "domain_hints": ["operations", "project-management"],
  "expected_value_type": "wikibase-entityid"
}
```

Output:

```json
{
  "candidates": [
    { "id": "R710002", "score": 0.88, "based_on": "P710" },
    { "id": "P710", "score": 0.61 }
  ],
  "selected": "R710002",
  "selection_confidence": 0.88
}
```

### 5.3 Claim compilation output (canonical)

```json
{
  "claim_id": "claim://sha256:<hash>",
  "subject": "Q100",
  "property": "R710002",
  "value": { "type": "wikibase-entityid", "content": "Q200" },
  "confidence": 0.86,
  "status": "asserted",
  "trace_refs": ["trace://sha256:<trace_root>"]
}
```

---

## 6. Confidence Strategy

SenTient’s outputs should provide:

* candidate confidence scores for Q and P/R/S
* overall selection confidence

Kristal compilation may compute a final confidence:

* `final = min(model_confidence, entity_conf, predicate_conf, evidence_conf)`
  or a weighted rule defined by validation policy.

---

## 7. Offline Mode

In offline mode, SenTient relies on:

* local Kristal `index` (labels/aliases)
* embedded charters (P/R/S)
* cached candidate stores

Rules:

* if resolution fails, emit placeholders and mark `needs_review`
* preserve trace logs for later enrichment when online

---

## 8. Output to Orgo DTOs (ACL)

If needed, compiled Kristal structures can be projected into Orgo DTOs:

* `EntityDTO` for Q-items
* relation/claim DTOs for P/R/S assertions

This mapping is documented in the Orgo interoperability spec.

---

## 9. Conformance

A pipeline conforms if it:

* resolves entities to QIDs with candidate lists + confidence,
* resolves predicates to P/R/S using charters,
* emits typed values,
* produces canonical claim objects suitable for Kristals,
* records non-deterministic decisions in trace logs.

