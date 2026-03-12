# Quickstart

Build a Kristal end-to-end: **Create → Validate → Publish → Activate → Query**.

## What you will produce
- **Exchange** (the portable bundle of curated knowledge)
- **Validation Report** (evidence that rules/profiles passed)
- **Runtime Pack** (query-optimized package for serving)
- (Optional) **Shard Manifest / Federation Manifest** (for scoped publishing + composition)

## Prerequisites
- Kristal compiler tooling installed (your internal build or release)
- A source snapshot (dataset / dump / crawl output)
- A place to publish artifacts (e.g., Konnaxion distribution store)
- Signing keys available (publisher/authority)

## 1) Create inputs (Claim-IR → Resolved Claim-IR)
Goal: normalize inputs into the canonical intermediate forms used by the pipeline.
- Generate **Claim-IR** from the source snapshot
- Resolve to **Resolved Claim-IR** (deterministic resolution)

## 2) Build an Exchange
Goal: produce a portable, content-addressed Kristal bundle.
- Compile an **Exchange**
- Ensure the Exchange includes the expected files (manifests + payloads)

See: [[Artifact-Exchange]]

## 3) Validate (fail-closed)
Goal: produce evidence you can ship and verify later.
- Run required validations (core + any profiles you claim)
- Produce **Validation Report(s)**
- Record validation references back into the relevant manifest(s)

See: [[Artifact-Validation-Report]]

## 4) Build a Runtime Pack (serving package)
Goal: generate the query-ready pack (indexes, bitmaps, policies, etc.).
- Compile a **Runtime Pack** from the Exchange
- Ensure policies are recorded (ordering, row-grouping, membership filters, bitmap)

See: [[Artifact-Runtime-Pack]]

## 5) Publish & distribute
Goal: push artifacts to the distribution layer so consumers can retrieve them.
- Publish Exchange + Runtime Pack + Validation Report(s)
- Publish/rotate authority material as needed
- Make the artifacts discoverable (index/registry in your distribution system)

See: [[Workflow-Publish-and-Distribute]]

## 6) Activate (and be able to rollback)
Goal: atomically switch consumers to the new Kristal.
- Activate the new Runtime Pack in the serving system
- Verify health checks / smoke queries
- Keep rollback ready (previous pack still retrievable + valid)

See: [[Workflow-Activate-Rollback-Downgrade]]

## 7) Query
Goal: confirm the Kristal works for real workloads.
- Run basic queries (sanity checks)
- Test pagination/capabilities if using the query contract

See: [[Query-Basics]] and [[Query-Pagination-and-Capabilities]]

---

## Optional: Shards & Federation (multi-publisher / scoped delivery)
If you publish multiple scoped Kristals (e.g., by domain/authority), use:
- **Shard Manifest** per shard
- **Federation Manifest** to compose shards deterministically
- **Authority Registry** for pinned trust policy data

See: [[Artifact-Shard-Manifest]], [[Artifact-Federation-Manifest]], [[Artifact-Authority-Registry]], [[Workflow-Federation-and-Curation]]

---

## Quick checks (when something fails)
- Validation is fail-closed: missing declared files/hashes/signatures should fail.
- Confirm you’re activating the intended Runtime Pack (ID/version).
- If federation fails: confirm authority registry reference and shard seals are present/valid.

## Next steps
- Read [[Concepts-and-Mental-Model]]
- Pick profiles you want to claim (see [[Artifacts]] and [[Workflows]])
- Set up CI gating (validation + determinism checks)