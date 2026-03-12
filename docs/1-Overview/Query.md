# Query

Kristal supports querying verified knowledge in a way that remains compatible with offline use, deterministic packaging, and explicit capability negotiation.

This page is an overview of **how querying works**, what you can expect from a Kristal runtime, and where to look for the concrete query contract.

---

## What “querying a Kristal” means

In practice, you query a **Runtime Pack** (built from an Exchange). The pack contains the indexes and policies needed to answer queries efficiently.

Typical query outputs are:
- entities / statements
- filtered subgraphs
- projections (e.g., “truthy” export)
- pagination streams

---

## Two modes you’ll see

### 1) Embedded / local query
- Runtime Pack is available locally (disk, offline device, edge node)
- Query engine reads directly from pack files
- Best for offline-correct workflows

### 2) Service / API query
- Runtime Pack is mounted in a serving system
- A query API provides access (capabilities negotiated)
- Best for platform and multi-tenant deployments

Both modes are expected to follow the same logical contract.

---

## Capability-driven contract

Kristal treats the query interface as a **contract**:
- runtimes declare what they support (capabilities)
- clients adapt (or fail early if requirements aren’t met)

Examples of capabilities:
- pagination type (cursor vs offset)
- supported query operators
- estimate availability (exact vs approximate)
- join/expansion constraints

See: [[Query-Pagination-and-Capabilities]]

---

## What is guaranteed vs implementation-defined

### Generally expected (stable expectations)
- deterministic, replayable results given the same pack + query
- fail-closed behavior when integrity is declared
- documented pagination semantics

### Often implementation-defined
- exact operator set (beyond core “get + filter + page” shapes)
- performance characteristics (latency, memory)
- secondary indexes available

---

## When federation affects querying

If you query a **Federation Manifest**, the runtime first composes shards deterministically (per composition policy), then serves results from the composed view.

This matters for:
- conflict resolution (which shard “wins”)
- authority precedence
- scope-based filtering

See: [[Workflow-Federation-and-Curation]]

---

## Where to go next

- Basics: [[Query-Basics]]
- Pagination & capabilities: [[Query-Pagination-and-Capabilities]]
- Artifact context: [[Artifact-Runtime-Pack]]
- Operational concerns: [[Operations-Observability-and-Troubleshooting]]