# Kristal Framework — Wiki

Kristal is a framework for producing, validating, distributing, and querying **deterministic knowledge artifacts** (“Kristals”) that remain **offline-correct** and **fail-closed** when integrity/trust signals are present.

This wiki is **product/usage oriented**: what Kristal does, the main artifacts, and the workflows. Technical specifications live in the docs repo and are linked when needed.

---

## Start here

- **Quickstart (end-to-end):** [Quickstart](Quickstart)
- **Glossary:** [Glossary](Glossary)
- **What is Kristal (mental model):** [Concepts & Mental Model](Concepts-and-Mental-Model)

---

## What you can do with Kristal

- Build a Kristal from sources (claims → resolved claims → exchange artifact)
- Validate outputs (schema/profile checks, integrity)
- Publish and distribute artifacts (versioned, rollbacks supported)
- Query locally or via services (capabilities + pagination)
- Compose multiple publishers (sharding + federation)

---

## Core concepts

- [Identity & Determinism](Identity-and-Determinism)
- [Trust, Authority & Signatures](Trust-Authority-and-Signatures)
- Shards & Federations (overview): [Federation & Curation](Workflow-Federation-and-Curation)

---

## Artifacts (what exists on disk / in distribution)

- [Exchange](Artifact-Exchange)
- [Runtime Pack](Artifact-Runtime-Pack)
- [Validation Report](Artifact-Validation-Report)
- [Shard Manifest](Artifact-Shard-Manifest)
- [Federation Manifest](Artifact-Federation-Manifest)
- [Authority Registry](Artifact-Authority-Registry)

---

## Workflows (how teams use it)

- [Build & Validate](Workflow-Build-and-Validate)
- [Publish & Distribute](Workflow-Publish-and-Distribute)
- [Activate, Rollback & Downgrade](Workflow-Activate-Rollback-Downgrade)
- [Subsets (Recipes)](Workflow-Subsets-Recipes)
- [Federation & Curation](Workflow-Federation-and-Curation)

---

## Query

- [Query Basics](Query-Basics)
- [Pagination & Capabilities](Query-Pagination-and-Capabilities)

---

## Operations

- [Release Strategy](Operations-Release-Strategy)
- [Observability & Troubleshooting](Operations-Observability-and-Troubleshooting)
- [Compatibility (v3 / v3.1)](Operations-Compatibility)

---

## FAQ

- [FAQ](FAQ)

---

## Tech specs (for implementers)

If you need exact schemas, normative rules, or conformance requirements, refer to the technical docs in the main repository (not the wiki).