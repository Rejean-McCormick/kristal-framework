# Introduction

Kristals are modular, offline-friendly **knowledge artifacts** designed to be consumed by AI systems (e.g., Sentient) and human experts as a structured, queryable foundation for reasoning.

A Kristal is not a document. It is a **portable knowledge graph**: a curated subset of entities, properties, and claims packaged so it can be downloaded, inspected, modified, and merged back into a broader ecosystem.

## Why Kristals

Modern AI systems are powerful at language, but weak at:
- **grounded factual recall** (hallucination risk),
- **verifiable provenance** (where did this fact come from?),
- **offline operation** (no network, no central database),
- **domain specialization** (experts need high-signal subsets, not the whole web).

Kristals address these limitations by turning knowledge into **versioned, testable modules**.

## Key Ideas

### 1) Modular knowledge packs
Instead of one massive global database, Kristals are distributed as:
- domain packs (e.g., botany, chemistry, local governance),
- curated collections aligned with fields of expertise (e.g., UNESCO field codes),
- task-specific slices (e.g., “plants of Québec”, “antibiotics interactions”).

### 2) Queryable like a database
Kristals are designed to be queried locally:
- by entity (`Q…` identifiers),
- by property (`P…` identifiers),
- by Orgo extensions (`R…` and `S…` identifiers),
- by semantic constraints (types, domains, relations).

The experience should feel like working with Wikidata, but offline and scoped.

### 3) Orgo-aligned enhanced Wikidata model
Kristals remain as close as possible to Wikidata/Wikibase alignment:
- **Q** for entities,
- **P** for compatible properties,

while supporting Orgo’s extensions:
- **R**: workflow refinements of existing properties,
- **S**: Orgo-specific properties beyond Wikidata.

This allows richer operational and domain-specific modeling without breaking the core alignment.

### 4) Detached trace logs (audit without weight)
Kristals stay lightweight.
Provenance and “how it was produced” live in detached trace logs, referenced by hashes:
- prompts, intermediate steps, confidence signals,
- evidence excerpts (or hashes + locators),
- compilation and validation reports.

### 5) Built for AI reasoning pipelines
Kristals are designed to work with AI ingestion and compilation:
- AI outputs produce claim IR,
- Sentient resolves entities/properties,
- validators enforce constraints,
- Kristals become a stable knowledge layer used during reasoning.

## What this repository contains

This repository defines:
- the Kristal format and schemas,
- the compilation pipeline (AI → IR → resolved claims),
- trace log standards,
- validation and merge behavior,
- reference architecture for offline packages and registries,
- interoperability with Orgo and Sentient.

## Quick vocabulary

- **Entity (Q)**: an item/concept node (Wikidata-aligned).
- **Property (P)**: a Wikidata property (when compatible).
- **Refinement (R)**: an Orgo refinement of a Wikidata property.
- **Orgo Property (S)**: an Orgo-specific property beyond Wikidata.
- **Claim**: a structured assertion connecting a subject to a value using P/R/S.
- **Kristal**: a packaged set of entities + claims + metadata.
- **Trace**: detached logs containing provenance and build details.

## Next

- Read the Kristal format specification: `docs/en/specification/format_kristal.md`
- See how claims are compiled: `docs/en/architecture/sentient_pipeline.md`
- Learn how to download and edit Kristals offline: `docs/en/tutorials/download_kristal.md`
