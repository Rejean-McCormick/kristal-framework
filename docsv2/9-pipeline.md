Voici **17 / 18** — le contenu proposé pour **`docs/9-pipeline.md`**.

---

# 9. End-to-End Pipeline — From Text to Executable Knowledge

## 9.1 Purpose

This document specifies the **normative pipeline** that turns unstructured text into **offline-executable, Wikidata-aligned knowledge**, and back into controlled natural language.

It defines **interfaces, invariants, and responsibilities** for each stage, enabling interchangeable implementations.

---

## 9.2 High-level flow

```
Sources (text, PDFs, web)
        ↓
   Claim-IR (schema-constrained)
        ↓
 Resolution + Validation
        ↓
  Kristal Exchange (canonical)
        ↓
  Kristal Runtime Pack (offline)
        ↓
 Queries / AI grounding / Apps
        ↓
 Architect (deterministic NLG)
```

The key rule: **probabilistic components are isolated upstream**; everything downstream is deterministic.

---

## 9.3 Stage A — Extraction to Claim-IR (AI / tools)

**Inputs**

* Text documents (web pages, PDFs, datasets)
* Optional hints (domain, language, target properties)

**Actors**

* LLMs (schema-constrained outputs)
* Classical extractors (NER/RE)
* Hybrid systems

**Output**

* `Claim-IR v0.1` (strict JSON Schema)

**Normative constraints**

* Output MUST validate against `3-claim-ir.schema.json`
* No prose, no summaries
* Uncertainty MUST be explicit (candidates, confidence, warnings)

**Rationale**
Claim-IR is the **safety membrane** between AI and knowledge.

---

## 9.4 Stage B — Resolution (Sentient)

**Responsibility**
Map surfaces to identifiers and normalize values.

**Functions**

* Surface → QID/PID candidate generation
* Scoring and ranking
* Literal normalization (time, quantity, units)

**Inputs**

* Claim-IR
* Local lexicons and subsets of **Wikidata**

**Outputs**

* Resolved Claim-IR (with candidates retained)
* Resolution warnings (non-fatal)

**Normative rules**

* No forced disambiguation without evidence
* No silent coercion
* Unresolved content MUST be preserved

---

## 9.5 Stage C — Validation (deterministic)

**Checks**

* Structural (schema)
* Type compatibility (property ↔ value)
* Constraint subsets aligned with Wikidata
* Provenance presence (profile-dependent)

**Outputs**

* Validated Kristal Exchange
* `validation_status ∈ {passed, partial, failed}`

**Invariant**

> If `failed`, the artifact MUST NOT be compiled to runtime.

---

## 9.6 Stage D — Kristal Exchange (canonical knowledge)

**Purpose**

* Interoperability
* Archival
* Mergeability

**Properties**

* Wikibase-aligned structure
* Content-addressed identity (`kristal_id`)
* Explicit unresolved sections (if any)

**Compatibility**

* Exportable to **Wikibase** JSON
* Exportable to RDF / JSON-LD

Kristal Exchange is the **source of truth**.

---

## 9.7 Stage E — Compilation to Runtime Pack

**Inputs**

* Kristal Exchange (validated)
* Compilation parameters (index policy, thresholds)

**Steps**

1. Assign dense integer IDs
2. Emit `(s,p,o)` triples
3. Build indexes by cardinality
4. Compute statistics
5. Optionally build membership filters
6. Write runtime manifest

**Output**

* Kristal Runtime Pack (offline executable)

**Determinism**
Same Exchange + same parameters ⇒ same Runtime Pack.

---

## 9.8 Stage F — Offline Querying

**Query profile**

* Triple-pattern first
* Set algebra (AND/OR/NOT)
* One-hop joins only

**Execution**

* Index lookup → subject sets
* Set operations
* No network, no SPARQL engine

**Performance note**
This stage aligns with set-oriented, bitmap-friendly execution models advocated in high-performance data engineering (e.g., work associated with **Daniel Lemire**).

---

## 9.9 Stage G — Deterministic Generation (Architect)

**Input**

* Query results (IDs + resolved values)
* Optional templates / grammars

**Output**

* Natural language text (multilingual possible)

**Rules**

* Generation occurs **after** validation
* No new facts introduced
* All statements trace back to Kristal claims

Architect turns **knowledge into text**, not the reverse.

---

## 9.10 Error handling and repair loop

At any stage before Exchange finalization:

1. Collect structured errors/warnings
2. Optionally invoke AI/human repair with constraints
3. Re-validate
4. Log each attempt

Repair is **incremental and minimal**.

---

## 9.11 Offline guarantees

The pipeline supports full offline execution once:

* Claim-IR schemas are local
* Wikidata subsets are cached
* No live endpoints are required

This enables:

* edge deployments
* regulated environments
* reproducible research

---

## 9.12 Responsibilities summary

| Stage      | Responsibility    | Deterministic |
| ---------- | ----------------- | ------------- |
| Claim-IR   | Proposal          | ❌             |
| Resolution | Mapping           | ⚠️            |
| Validation | Acceptance        | ✅             |
| Exchange   | Canonical storage | ✅             |
| Runtime    | Execution         | ✅             |
| Architect  | Rendering         | ✅             |

---

## 9.13 Summary

This pipeline:

* isolates AI uncertainty,
* preserves Wikidata compatibility,
* enables offline execution,
* and supports deterministic generation.

It is the **operational backbone** of Kristal.

---

**Next document:** `10-implementation-notes.md`
