Voici **2 / 18** — le contenu proposé pour **`docs/0-overview.md`**.

---

# 0. Overview — Kristal in one view

## 0.1 What problem Kristal addresses

Kristal addresses a gap between **open encyclopedic knowledge** and **practical execution**, especially in offline or deterministic AI contexts.

Today:

* **Wikidata** provides an unparalleled open knowledge graph, but is optimized for *global, online, SPARQL-based* access.
* AI systems (LLMs) are excellent at producing text, but poor at producing **verifiable, reusable, structured knowledge**.
* Offline systems (edge, regulated, constrained environments) cannot rely on heavy endpoints or probabilistic answers.

Kristal defines a **missing middle layer**:

> a portable, executable, Wikidata-aligned unit of knowledge.

---

## 0.2 What a Kristal is (and is not)

### Kristal **is**

* A **block of encyclopedic knowledge**
* **Structured**, **verifiable**, **mergeable**
* **Aligned with Wikidata identifiers** (QIDs, PIDs)
* **Executable offline**
* **AI-consumable and AI-producible**

### Kristal **is not**

* A document
* Free text
* A competing ontology
* A SPARQL endpoint
* A black-box AI output

A Kristal is closer to a **compiled artifact** than to a text or database dump.

---

## 0.3 Core idea: separation of concerns

Kristal is built on a strict separation between:

| Layer                    | Purpose          | Properties                    |
| ------------------------ | ---------------- | ----------------------------- |
| **Claim-IR**             | Bridge from LLMs | Strict schema, evidence-first |
| **Kristal Exchange**     | Interoperability | Wikibase-aligned JSON         |
| **Kristal Runtime Pack** | Execution        | Indexed, offline, fast        |

This separation is deliberate:

* Exchange formats prioritize **correctness and compatibility**
* Runtime formats prioritize **performance and determinism**

This mirrors how compilers separate **source code** from **machine code**.

---

## 0.4 Relationship to Wikidata and Wikibase

Kristal is **not a fork** of Wikidata.

Instead:

* QIDs and PIDs are reused as-is
* Wikibase semantics (claims, qualifiers, references) are preserved
* Export to Wikibase JSON / RDF / JSON-LD is always possible

Kristal may be understood as:

> a **portable, executable subset** of Wikidata semantics.

This design ensures alignment with:

* Wikibase
* existing Wikidata tooling
* future initiatives such as Abstract Wikipedia

---

## 0.5 Offline-first by design

Kristal assumes:

* no network access
* no SPARQL service
* no LLM at runtime

Instead, Kristal relies on:

* compact dictionaries (IDs → integers)
* columnar or triple-table storage
* bitmap / list-based set operations
* constrained query profiles

This approach is informed by:

* compressed RDF research (HDT, columnar RDF)
* bitmap index systems
* performance-oriented data engineering

The result: **predictable performance** in constrained environments.

---

## 0.6 Deterministic AI integration

Kristal enforces a strict rule:

> **LLMs never produce prose directly into Kristals.**

Instead:

1. LLMs output **Claim-IR** (structured, schema-constrained)
2. Claims are resolved, validated, and grounded
3. Only validated Kristals may be used downstream
4. Prose is generated *after* validation

This enables:

* reproducibility
* inspection
* controlled generation
* avoidance of hallucinations as “knowledge”

---

## 0.7 Provenance and integrity

Every Kristal can carry:

* explicit evidence (quotes, sources)
* content-addressable identifiers (hashes)
* optional signatures or attestations

This makes Kristals suitable for:

* regulated environments
* scientific workflows
* long-term archival
* collaborative merging

Kristal treats provenance as **first-class**, not metadata.

---

## 0.8 Intended use cases

Kristal is designed for:

* Offline encyclopedic search
* AI grounding datasets
* Deterministic content generation
* Knowledge packs for education, research, governance
* Domain-specific subsets (medicine, law, organizations)

It is explicitly **not** designed for:

* ad-hoc natural language Q&A
* real-time global graph mutation
* open-ended inference engines

---

## 0.9 Status and evolution

This specification:

* is **proposal-level**
* is intended for technical review and prototyping
* prioritizes clarity over completeness

Backward compatibility is a design constraint:

* newer Kristal versions must degrade cleanly
* export to Wikidata-compatible formats remains mandatory

---

## 0.10 Reading guide

Recommended order:

1. `1-standard.md` — scope and positioning
2. `2-prior-art.md` — state of the art and credits
3. `3-claim-ir.md` — LLM bridge
4. `4-kristal-exchange.md` — canonical format
5. `5-kristal-runtime.md` — offline execution model

---

**Next document:** `1-standard.md`
