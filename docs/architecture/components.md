# Components (Kristal Framework)

## Status
DRAFT

## 1. Overview

Kristal Framework is composed of modular components that together support:
- generating Kristals from AI/human inputs,
- resolving identifiers (Q/P/R/S),
- validating and packaging offline knowledge packs,
- querying and slicing,
- merging and publishing updates.

This document describes the reference architecture. Implementations may combine components (monolith) or split them (services), as long as interfaces are preserved.

---

## 2. High-Level Component Map

**Core build path**
1) Ingest (IR / WikidataGraph)  
2) Resolve (Q/P/R/S)  
3) Compile (claims)  
4) Validate (policies)  
5) Package (Kristal + optional traces + indexes)  

**Runtime path**
1) Load Kristal(s)  
2) Query / Slice  
3) Reason (AI)  
4) Update / Merge  

---

## 3. Components

### 3.1 Kristal Core Model
**Responsibility**
- Defines the in-memory representation of:
  - `manifest`
  - `charters`
  - `qitems`
  - `claims`
  - optional `index`

**Key properties**
- stable canonical ordering
- deterministic serialization
- schema validation

**Inputs**
- compiler output, registry downloads

**Outputs**
- serialized `*.kristal.json`

---

### 3.2 Charter Manager (Property Catalog)
**Responsibility**
- Loads Orgo charters (P/R/S catalogs)
- Merges layered charts:
  - `general`
  - `family`
  - optional `domain`
- Builds a lookup lexicon:
  - `label/alias → property ID`
  - `property ID → constraints`

**Why it exists**
- compilation and validation require a shared property vocabulary.

**Outputs**
- property definitions for:
  - predicate resolution
  - type checking
  - constraint enforcement

---

### 3.3 Entity Resolver (Q Resolver)
**Responsibility**
- Resolves text mentions to QIDs
- Provides candidate lists and confidence scores

**Typical submodules**
- lexical candidate generator (labels/aliases)
- optional semantic reranker (embeddings)
- disambiguation using local context (neighbor claims)

**Inputs**
- mention text
- optional context (question, evidence excerpt, surrounding entities)
- local indexes (from Kristal packs)

**Outputs**
- selected `Q...` or placeholder `Q_tmp_*`
- candidate list with scores (traceable)

---

### 3.4 Predicate Resolver (P/R/S Resolver)
**Responsibility**
- Maps predicate intents to:
  - `P...` (Wikidata-aligned)
  - `R...` (refinement)
  - `S...` (Orgo-specific)

**Inputs**
- predicate text
- expected value type (if available)
- domain chart and property constraints

**Outputs**
- selected property ID
- candidate list + decision trace

---

### 3.5 Value Typer
**Responsibility**
- Converts raw object values into typed values:
  - entity ids, strings, dates, quantities, coords, urls

**Inputs**
- raw object text / tokens
- predicate constraints (value_type)
- Q-resolution results (if object is an entity)

**Outputs**
- typed value object for claim compilation

---

### 3.6 Claim Compiler
**Responsibility**
- Produces canonical Kristal claims:
  - subject + property + typed value
  - qualifiers/references (optional)
  - confidence and status
  - trace references

**Key guarantees**
- deterministic canonicalization
- stable `claim_id` hashing

**Outputs**
- `claims[]` list for Kristal core

---

### 3.7 Validator & Scrutinizers
**Responsibility**
- Enforces correctness and policy:
  - schema-level validation
  - value type compatibility
  - required evidence rules
  - constraint checks (cardinality, allowed values)
  - duplicate detection
  - contradiction detection (optional)

**Outputs**
- claim status transitions:
  - `asserted`, `inferred`, `needs_review`, `rejected`, `conflicted`
- validation report for packaging and merge review

---

### 3.8 Trace Manager (Detached Logs)
**Responsibility**
- Stores and retrieves trace bundles:
  - LLM runs, Sentient resolution, validation details
  - evidence excerpts or hashes + locators
- Produces content-addressed IDs:
  - `trace://sha256:*`
  - `source://sha256:*`

**Outputs**
- `trace_root` refs
- per-claim `trace_refs[]`

---

### 3.9 Index Builder (Offline Acceleration)
**Responsibility**
- Builds local indexes for:
  - label/alias lookup (Q resolution)
  - property lexicon (P/R/S resolution)
  - optional embeddings

**Outputs**
- `index` block in Kristal core (optional)
- or sidecar index files in a pack

---

### 3.10 Query Engine
**Responsibility**
- Executes local queries over:
  - `qitems[]` + `claims[]`
- Supports:
  - triple-pattern matching
  - filters on confidence/status/context
  - joins and multi-hop traversals (bounded)

**Outputs**
- result bindings (entities, claims)
- slice-ready entity sets

---

### 3.11 Slicer (Subset Packager)
**Responsibility**
- Produces a new Kristal from:
  - query results
  - closure rules (outgoing/bidirectional)
  - depth constraints
- Ensures closure completeness:
  - includes referenced QIDs and required ontology parents (optional)

**Outputs**
- derived Kristal + lineage metadata

---

### 3.12 Merge Engine
**Responsibility**
- Merges Kristals and resolves conflicts:
  - deduplicate by `claim_id` or canonical triple identity
  - keep context separation (core/inferred/contested/local)
  - create merge reports for review

**Outputs**
- merged Kristal
- merge report (traceable)

---

### 3.13 Registry (Distribution Layer)
**Responsibility**
- Stores published Kristals and charter refs
- Supports:
  - discovery by domain tags (e.g., UNESCO)
  - versioning
  - downloads and slice endpoints
  - optional signatures

**Outputs**
- stable distribution endpoints
- metadata catalog
- optional patch/diff delivery

---

## 4. Runtime Consumption (AI / App)

At runtime, an AI agent typically uses:
- **Query Engine** to retrieve relevant claims
- **Index** to map text to candidate entities/properties
- **Contexts** to filter trust levels
- **Trace refs** when audit is requested

---

## 5. Recommended Deployments

### 5.1 Offline workstation
- local Kristal store + query engine
- local indexes
- optional local trace bundle storage

### 5.2 Server-backed registry
- registry API
- compilation + validation workers
- storage for Kristals + traces
- search index for discovery

### 5.3 Hybrid agent
- offline packs cached locally
- registry sync when online
- slice on-demand by domain or query

---
