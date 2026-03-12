# Artifact: Runtime Pack

A **Runtime Pack** is the execution-optimized artifact derived from an Exchange. It is designed for **fast activation** and **predictable query performance** while preserving Kristal’s core guarantees (determinism, verifiability, safe rollout/rollback).

---

## What it’s for

Use a Runtime Pack when you need to:
- activate a Kristal into a runtime (local or service)
- serve queries efficiently (indexes, layouts, precomputed structures)
- standardize the “portable performance surface” (recorded policies)
- support safe distribution (versioned packs, atomic activation, rollback)

---

## When it exists

A Runtime Pack is typically produced:
- after an Exchange is built and validated
- when you want to deploy/query the Kristal at scale
- when you need stable, recorded policies for performance-relevant choices (ordering, grouping, filters, bitmap format, etc.)

---

## What it contains (at a high level)

A Runtime Pack generally includes:
- a **manifest** (what’s inside, how it was built, what policies were used)
- **data files** (optimized layouts; e.g., columnar partitions)
- **indexes / auxiliary structures** (bitmaps, membership filters, etc.)
- **query contract info** (declared capabilities, limits, pagination behavior)
- optional **integrity and signature material** (for verification and trust)

This wiki keeps it conceptual; for exact fields see the technical docs.

---

## Who produces it / who consumes it

**Produced by**
- compiler/build pipeline (“pack builder” step)

**Consumed by**
- activation tooling (installer/launcher)
- query engines
- operational tooling (rollout, rollback, monitoring)

---

## Verification expectations

Consumers typically verify:
- pack files referenced by the manifest are present
- declared hashes match the files
- signatures are valid (when present and required by policy)
- the pack references the intended Exchange (correct lineage)

If verification is enabled and declared signals are present, failure should be treated as **fail-closed**.

---

## Operational behavior

Runtime Packs are designed to support:
- **atomic activation** (activate a complete pack or not at all)
- **rollback/downgrade** (switch back to a known-good pack)
- **parallel installs** (multiple packs side-by-side, select active one)
- **compatibility controls** (v3 vs v3.1 surface expectations)

See: [Activate, Rollback & Downgrade](Workflow-Activate-Rollback-Downgrade)

---

## Related pages

- Exchange (source artifact): [Artifact: Exchange](Artifact-Exchange)
- Query behavior: [Query Basics](Query-Basics)
- Ops: [Release Strategy](Operations-Release-Strategy)

---

## Tech details

- Schema + normative behavior: runtime-pack manifest spec in the technical docs (`kristal-docs-v3/02-schemas/runtime-pack-manifest.schema.json`)
- Example manifests: `kristal-docs-v3/10-examples/runtime-pack-manifest.example.json`