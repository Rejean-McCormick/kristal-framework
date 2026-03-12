# Artifact: Shard Manifest

A **Shard Manifest** describes an **authoritative slice** of knowledge for a specific **scope** (e.g., a domain/subdomain/time window), plus the information consumers need to **verify** and **compose** it.

It’s used when you want:
- multiple publishers (different authorities) in different scopes
- safe, incremental updates (only the changed shard gets a new identity)
- federation/curation without merging everything into one monolith

## What it is (in one sentence)
A Shard Manifest is a **verifiable pointer** to a shard Exchange (or shard bundle), annotated with **scope**, **validation evidence**, and **authority**.

## When it exists
You create a Shard Manifest when:
- a publisher produces a domain-scoped Exchange intended to be reused or composed
- you need explicit authority signaling for a slice of knowledge
- you plan to build a Federation Root over multiple shards

If you don’t use sharding/federation, you may never produce this artifact.

## What it typically contains (high-level)
- **Identity**
  - shard ID (content-addressed)
  - created timestamp (non-identity)
- **Scope**
  - domain / subdomain / time slice (and optional tenant/env)
- **Inputs**
  - the source snapshot reference
  - recipe/subset reference (if applicable)
- **Validation references**
  - links/hashes to Validation Reports that attest required checks passed
- **Integrity**
  - hashes and/or signatures over a stable target
- **Authority**
  - who sealed it (authority identifier + key id)

## How consumers use it
Consumers typically:
1) **Fetch** the shard Exchange referenced by the manifest
2) **Verify integrity**
   - if integrity is declared → fail closed on mismatch
3) **Check validations**
   - ensure required validation reports exist and passed
4) **Check authority (if enabled)**
   - confirm the authority is allowed for the shard’s scope (via pinned policy/registry)
5) **Expose it for composition**
   - federation roots can reference this shard as a building block

## Relationship to other artifacts
- **Shard Manifest → Exchange**
  - the shard’s actual knowledge content is in an Exchange (or shard bundle)
- **Shard Manifest ↔ Validation Report**
  - validation reports prove the shard passed required checks
- **Shard Manifest ↔ Authority Registry**
  - the registry defines which authorities are acceptable per scope (when used)
- **Shard Manifest → Federation Manifest**
  - a federation root composes multiple shard manifests

## Operational notes
- Updating one shard creates a **new shard ID**, but does not force reissuing other shards
- Prefer stable scope definitions (so consumers can route trust and caching predictably)
- Keep validation evidence pinned and replayable for audits

## Next pages
- Artifact: Federation Manifest
- Artifact: Authority Registry
- Workflows: Federation & Curation