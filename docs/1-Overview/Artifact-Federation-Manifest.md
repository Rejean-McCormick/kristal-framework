# Artifact: Federation Manifest

A **Federation Manifest** defines a **deterministic composition** of multiple shards (or domain-scoped Exchanges) into a single *federated* Kristal view, without rewriting the shards themselves.

Use it when you want to:
- Combine multiple publishers/domains into one coherent “product”
- Keep shard ownership and authority intact
- Update one domain independently (new shard) without rebuilding everything

## What it’s for

A Federation Manifest is the “composition root” for knowledge:

- **Curated composition**: declare which shards are included
- **Deterministic merge rules**: define precedence and conflict handling
- **Trust-aware consumption**: allow consumers to verify shards and apply authority policy
- **Portable distribution**: publish a single federated entrypoint that points to many shards

## What it contains (conceptually)

A Federation Manifest typically includes:

- **Federation identity** (content-addressed)
- **Shard references**:
  - which shards are included
  - how to locate their shard manifests
  - integrity references (hashes and/or seals) if you want self-contained verification
- **Composition policy**:
  - ordering and precedence rules
  - conflict resolution rules
- **Optional trust requirements** (hints for consumers)
- **Optional authority registry reference** (pinned trust policy data)
- **Publisher metadata and signatures** (who publishes this federation root)

## Who produces and who consumes it

**Produced by**
- A curator / federation publisher (human or automated system)

**Consumed by**
- Stores and activation pipelines (to publish/activate the federated view)
- Verifiers/gates (to ensure the federation is acceptable for a deployment)
- Query services that want a single entrypoint to a multi-shard world

## Lifecycle placement

Federation fits into the standard flow, but operates at the “composition layer”:

1. Publish shards (independently, per domain)
2. **Create/publish Federation Manifest**
3. Activate federation root in a store
4. Query the federated view (resolving shard set + applying composition rules)

## Key properties / guarantees

- **Shard isolation**: updating one shard does not mutate others
- **Explicit conflicts**: composition rules make precedence deterministic
- **Trust is explicit**: authority is checked via policy (often pinned registry data)
- **Offline-correct**: federation can be verified and executed without network, if artifacts are present

## Related pages

- [[Artifact-Shard-Manifest]]
- [[Artifact-Authority-Registry]]
- [[Workflow-Federation-and-Curation]]
- [[Operations-Compatibility]]

## Tech details (links)

- Spec: `00-overview/sharding-and-federation-upgrade.md`
- Schema: `02-schemas/exchange-federation-manifest.schema.json`
- Example: `10-examples/exchange-federation-manifest.example.json`