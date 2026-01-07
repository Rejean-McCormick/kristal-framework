Voici **1 / 18** — le contenu proposé pour **`docs/README.md`**.

---

# Kristal — Offline-first Encyclopedic Knowledge Blocks

**Status:** Draft specification (v0.1)
**Audience:** Wikidata / Wikibase / W3C community, knowledge engineering, AI systems

## What is Kristal?

**Kristal** is a standard for **encyclopedic knowledge blocks**: small, verifiable, reusable units of structured knowledge, designed to be:

* **Aligned with Wikidata / Wikibase**
* **Executable offline** (no SPARQL endpoint, no LLM dependency)
* **Verifiable and traceable** (explicit evidence and provenance)
* **AI-ready** (usable as grounding substrates, not prose)

Kristals are not documents and not free text.
They are **compiled knowledge artifacts**.

---

## Why Kristal?

Existing knowledge graphs (notably Wikidata) excel at **global openness and interlinking**, but:

* are hard to **execute offline**
* require **full SPARQL generality** for many use cases
* are not optimized for **deterministic AI grounding**
* mix **exchange format** and **execution format**

Kristal addresses this by:

1. preserving full **Wikidata semantic compatibility**
2. introducing a **clear separation** between:

   * exchange (interop)
   * execution (performance)
3. defining a **deterministic compilation pipeline** from text → knowledge → runtime

---

## Core Design Principles

### 1. Wikidata alignment, not replacement

* QIDs and PIDs are first-class
* Kristals can be exported to Wikibase JSON, RDF, or JSON-LD
* No competing ontology

### 2. Offline-first execution

* No SPARQL endpoint required
* Queries expressed as constrained patterns (set operations)
* Runtime based on compact indexes and dictionaries

### 3. Deterministic AI integration

* LLMs produce **structured Claim-IR**, not prose
* Claims are compiled, validated, and grounded
* Natural language is generated *after* validation

### 4. Traceability and integrity

* Every claim can carry evidence
* Content-addressable IDs (hashes)
* Optional signatures and attestations

---

## The Kristal Stack (high-level)

```
Text / Documents
        ↓
   Claim-IR (LLM-friendly, structured)
        ↓
   Kristal Exchange (Wikibase-aligned JSON)
        ↓
   Kristal Runtime Pack (offline, indexed)
        ↓
 Queries / AI / Applications
```

---

## Scope (v0.1)

**Included**

* Claim-IR specification (LLM → knowledge bridge)
* Kristal Exchange format (Wikibase-aligned)
* Kristal Runtime Pack (offline execution)
* Query profile (no full SPARQL)
* Validation, provenance, integrity
* End-to-end pipeline (Sentient → Kristal → Architect)

**Explicitly excluded**

* Full SPARQL engine
* Centralized hosting requirements
* Proprietary ontologies
* Black-box AI reasoning

---

## Relationship to existing initiatives

Kristal is designed to be **compatible with**, and complementary to:

* Wikidata
* Wikibase
* W3C community work on JSON-LD, RDF, and Linked Data
* Abstract Wikipedia / structured content generation
* Offline knowledge systems and edge deployments

Kristal can be seen as a **portable, executable subset** of Wikidata semantics.

---

## Document set overview

This repository contains **18 documents**, ordered to support:

1. understanding
2. specification
3. execution
4. implementation

Start here → `0-overview.md`
Then → `1-standard.md` and `2-prior-art.md`

---

## Status

This is a **proposal-level specification** intended for:

* technical discussion
* prototyping
* possible submission to standards or community groups

Feedback, critique, and experimentation are explicitly encouraged.

---

**Next document:** `0-overview.md`
