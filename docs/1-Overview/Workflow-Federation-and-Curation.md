# Workflow: Federation & Curation

Federation lets you **compose multiple authoritative shards** into a single consumable view without forcing a monolithic rebuild. Curation is the human/process layer that decides **which shards to include** and **how conflicts are resolved**.

## Goal
Produce a **Federation Manifest** that:
- references a set of shard artifacts (by identity)
- declares deterministic composition rules
- can be verified offline (integrity + authority checks)

## When you use this
Use federation when:
- different publishers are authoritative for different scopes (domains/subdomains)
- you want independent update cadence per scope
- you need a curated “best view” built from multiple sources

If you have only one publisher and one Exchange, federation is optional.

## Inputs
- A set of **Shard Manifests** (or shard Exchange references) with known identities
- Optional **Authority Registry** (pinned policy data describing who is valid for what scope)
- A curation decision (which shards, which precedence rules)

## Outputs
- **Federation Manifest** (the root composition artifact)
- Optional: a “federation runtime pack” or index for distribution (deployment choice)

## Step-by-step

### 1) Collect candidate shards
- Gather shard identities from publishers you intend to include
- Ensure each shard declares its scope clearly
- Ensure you can fetch the shard artifacts offline (or pin them in distribution)

### 2) Verify each shard (fail-closed)
For every shard:
- verify declared integrity (hash/signatures)
- verify required validations passed (validation reports)
- if authority policy is enabled:
  - check the shard’s authority is permitted for the shard scope (via pinned registry)

Only verified shards should proceed to composition.

### 3) Define composition policy (deterministic)
Decide rules such as:
- precedence by authority class (recognized > personal)
- precedence by scope specificity (subdomain overrides domain)
- tie-breaking rules (publisher priority list, newest shard, explicit deny/allow)

The key requirement: the same input shards + same policy => same composed result.

### 4) Build the Federation Manifest
Include:
- federation identity metadata
- the list of shards (with references + integrity hints)
- the composition policy
- optional trust hints (if you publish guidance to consumers)

### 5) Sign / seal the federation root
- The curator/publisher signs the federation manifest
- This signature attests: “this is the intended composition”
- It does **not** replace shard authority—shards remain authoritative for their scopes

### 6) Publish and distribute
Distribute:
- Federation Manifest
- referenced shard artifacts (or resolvable references)
- pinned Authority Registry (if used)

## Operational guidance
- Treat federation as an “index” over shards; keep it small and frequently updatable
- Prefer pinned policy data for offline verification (avoid remote calls during verification)
- Keep rollback paths: a federation root can move back to a previous shard set safely

## Common pitfalls
- Composing shards without verifying each shard (breaks trust guarantees)
- Non-deterministic conflict resolution (consumers get inconsistent answers)
- Using “trust hints” as policy (hints are not enforcement)
- Not pinning policy/registry artifacts (verification depends on mutable data)

## Next pages
- Artifact: Federation Manifest
- Artifact: Authority Registry
- Artifact: Shard Manifest
- Workflow: Activate, Rollback & Downgrade