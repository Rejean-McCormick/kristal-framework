# Artifacts

Kristal produces a small set of standard artifacts that can be verified, distributed, and composed. This page is a map of **what exists**, **what it’s for**, and **when you’ll see it**.

> This wiki stays high-level. For exact fields and schemas, follow the “Tech details” links on each artifact page.

---

## Core artifacts (most workflows)

### Exchange
**What it is:** the portable bundle of curated knowledge for a build.  
**When you get it:** every time you compile a Kristal.  
**Used by:** validators, distributors, pack builders, offline consumers.

See: [[Artifact-Exchange]]

### Validation Report
**What it is:** evidence that required checks passed (core rules + any profiles).  
**When you get it:** whenever you validate an Exchange or derived artifact.  
**Used by:** offline verification, CI gating, federation consumers.

See: [[Artifact-Validation-Report]]

### Runtime Pack
**What it is:** a query-optimized package built from an Exchange (indexes, bitmaps, policies).  
**When you get it:** when you prepare a Kristal for serving/query.  
**Used by:** query engines and serving systems.

See: [[Artifact-Runtime-Pack]]

---

## Federation artifacts (multi-publisher / scoped delivery)

### Shard Manifest
**What it is:** a manifest that describes a scoped shard (domain/time/authority), with integrity and validation references.  
**When you get it:** when you publish scoped Kristals (by domain, authority, tenant, etc.).  
**Used by:** federation builders/consumers.

See: [[Artifact-Shard-Manifest]]

### Federation Manifest
**What it is:** a deterministic composition of multiple shards into one “federated Kristal”.  
**When you get it:** when a curator publishes a composed view across multiple shards/publishers.  
**Used by:** consumers who want a single entry point with policy-driven composition.

See: [[Artifact-Federation-Manifest]]

### Authority Registry
**What it is:** pinned trust policy data (trust roots + rules) used to decide which authorities are acceptable for which scopes.  
**When you get it:** when you operate multi-authority or multi-publisher systems.  
**Used by:** federation verification and policy enforcement.

See: [[Artifact-Authority-Registry]]

---

## Typical dependency graph (conceptual)

- **Exchange** → validated by → **Validation Report**
- **Exchange** → compiled into → **Runtime Pack**
- **Shard Manifest** references → shard Exchange + validation evidence + authority seals
- **Federation Manifest** references → shard manifests (and optionally their seals)
- **Authority Registry** guides → verification and acceptance rules for shards/federations

---

## How to choose what you need

- If you just want “a Kristal you can query”: **Exchange + Runtime Pack (+ Validation Report)**
- If you need offline trust / audit: **add Validation Reports + signatures**
- If you need multiple publishers or scoped sources: **add Shards + Federation + Authority Registry**