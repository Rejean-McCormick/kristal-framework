Voici **5 / 18** — le contenu proposé pour **`docs/3-claim-ir.md`**.

---

# 3. Claim-IR — Structured Claims Intermediate Representation (v0.1)

## 3.1 Purpose

**Claim-IR** is the **only format** that AI systems (LLMs, extractors) are allowed to produce directly in the Kristal stack.

Its role is to form a **hard, deterministic boundary** between:

* probabilistic extraction (LLMs),
* and verifiable, executable knowledge (Kristals).

> Claim-IR is **not knowledge**.
> It is a **proposal of claims**, with evidence, candidates, and uncertainty.

---

## 3.2 Design principles

Claim-IR is designed to be:

1. **Schema-constrained**
   Must be generated under strict JSON Schema (no free-form JSON).

2. **Evidence-first**
   Claims should be anchored to sources whenever possible.

3. **Wikidata-aligned but tolerant**
   QIDs/PIDs are preferred, but unresolved surfaces and candidates are allowed.

4. **Degradable, never lossy**
   Missing resolution produces partial output, not rejection or hallucination.

5. **Minimal**
   No prose, no narrative, no inference.

---

## 3.3 Position in the Kristal pipeline

```
Text / Documents
        ↓
   Claim-IR  ←─── produced by LLM / extractor
        ↓
   (resolution, validation, compilation)
        ↓
   Kristal Exchange
```

LLMs **never** produce Kristal Exchange or Runtime Packs directly.

---

## 3.4 Core structure (conceptual)

A Claim-IR document contains:

* **subject**
* **claims[]**
* optional **document metadata**
* optional **warnings**

### Subject

Represents *what the document is primarily about*.

* Prefer a `qid` if confidently resolved
* Otherwise use a `surface` + `candidates[]`
* Ambiguity is allowed and explicit

Example (conceptual):

```json
"subject": {
  "surface": "Ada Lovelace",
  "lang": "en",
  "candidates": [
    { "id": "Q7259", "score": 0.94 }
  ]
}
```

---

### Claims

Each claim represents **one atomic assertion**:

* predicate (property)
* object (value)
* optional qualifiers
* evidence
* confidence
* alternatives (optional)

A claim must be understandable **without context**.

Conceptually:

```json
{
  "predicate": { "pid": "P569" },
  "object": { "kind": "time", "value": { "iso8601": "1815-12-10" } },
  "evidence": [...],
  "confidence": 0.9
}
```

---

## 3.5 Objects and datatypes

Claim-IR supports the same **core value types** as Wikidata:

| Kind               | Meaning                          |
| ------------------ | -------------------------------- |
| `item`             | Linked entity (QID or candidate) |
| `string`           | Plain string                     |
| `monolingual_text` | Text with language               |
| `quantity`         | Numeric value with optional unit |
| `time`             | Time value with precision        |
| `coord`            | Geographic coordinates           |
| `url`              | URI                              |

Type correctness is enforced **at compilation**, not generation time.

---

## 3.6 Evidence model

Evidence is **first-class** in Claim-IR.

Each claim may include one or more evidence objects:

* source URL or ID
* optional quote
* optional offsets/page numbers

Evidence is:

* required for “grounded” profiles
* optional but strongly encouraged otherwise

Claim-IR does **not** assess truth — it records **support**.

---

## 3.7 Candidates and ambiguity

Claim-IR explicitly models ambiguity:

* `subject.candidates[]`
* `predicate.candidates[]`
* `object.value.candidates[]` (for item values)

This avoids:

* forced disambiguation
* hallucinated certainty

Resolution is delegated to **Sentient** or downstream tooling.

---

## 3.8 Confidence scores

Each claim may include a `confidence ∈ [0,1]`.

Rules:

* Confidence expresses **extraction confidence**, not truth.
* Compilers may down-rank or flag low-confidence claims.
* Confidence does not bypass validation or evidence rules.

---

## 3.9 Warnings (non-fatal issues)

Claim-IR may include warnings such as:

* unresolved QID/PID
* missing evidence
* low confidence
* ambiguous subject

Warnings:

* never block compilation by themselves
* are propagated to Kristal metadata

---

## 3.10 Conformance levels

Claim-IR supports graded conformance:

| Level | Meaning                          |
| ----- | -------------------------------- |
| L0    | Structurally valid               |
| L1    | Compilable (types + structure)   |
| L2    | Grounded (evidence present)      |
| L3    | Publishable (passes constraints) |

Compilation may proceed from **L1+**, but L3 is recommended for distribution.

---

## 3.11 Explicit non-goals

Claim-IR does **not**:

* encode inference or reasoning
* perform disambiguation
* guarantee correctness
* include prose or summaries
* attempt to mirror RDF directly

Its job is **proposal, not truth**.

---

## 3.12 Relationship to Wikidata

Claim-IR is **Wikidata-aware but not Wikidata-complete**.

* Uses QIDs and PIDs where possible
* Preserves unresolved surfaces
* Maps cleanly to Wikibase claims during compilation

It is intentionally **simpler** than Wikibase JSON.

---

## 3.13 Summary

Claim-IR is the **keystone** of the Kristal architecture:

* it constrains AI output,
* preserves uncertainty,
* enforces structure,
* and enables deterministic compilation.

Without Claim-IR, Kristal collapses back into text.

---

**Next document:** `3-claim-ir.schema.json`
