# Getting Started

Kristal is a deterministic, offline-correct knowledge artifact system: you **build** a Kristal from source data, **validate** it, **publish/distribute** it, then **query** it locally or through a platform.

## What you can do with Kristal
- Package a curated knowledge snapshot you can reproduce and verify
- Run queries offline (or through a service) with predictable behavior
- Update safely (roll forward, rollback, or downgrade) without breaking consumers
- Federate multiple publishers (shards + federation roots) when needed

## Core building blocks (mental model)
- **Exchange**: the published “unit” of knowledge + metadata
- **Runtime Pack**: optimized runtime representation for query/render
- **Validation Report**: proof that the artifact passed declared checks
- **Trust/Authority**: who is allowed to publish for a given scope (optional)

## Typical workflow (end-to-end)
1) **Build**
   - Ingest source data
   - Produce Claim-IR / Resolved Claim-IR (implementation detail)
   - Produce an **Exchange**
   - Produce a **Runtime Pack** (optional but common)

2) **Validate**
   - Run required validations (schema + profiles)
   - Produce a **Validation Report**
   - Fail-closed: if integrity is declared and doesn’t verify → reject

3) **Publish & Distribute**
   - Publish artifacts (Exchange, Runtime Pack, Validation Report)
   - Distribute via your mechanism (repo, object store, Konnaxion, etc.)

4) **Activate**
   - Consumers fetch and activate a specific version
   - Keep rollback/downgrade paths available when operating at scale

5) **Query**
   - Query locally (offline) or through a service
   - Use declared capabilities and pagination patterns

## If you’re new: do this first
- Read **What is Kristal**
- Read **Artifacts → Exchange** and **Artifacts → Runtime Pack**
- Follow **Workflows → Build & Validate** and **Workflows → Publish & Distribute**
- Then go to **Query → Query Basics**

## Where to find technical details
The wiki stays high-level. For schemas/specs/examples, use the docs repository:
- Core specs
- JSON schemas
- Examples / test vectors

## Next pages
- Quickstart
- What is Kristal
- Artifacts (Exchange, Runtime Pack, Validation Report)
- Workflows (Build/Validate, Publish/Activate, Rollback)
- Query (Basics, Pagination & Capabilities)
- FAQ / Glossary