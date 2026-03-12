# Artifact: Exchange

An **Exchange** is the primary Kristal bundle you publish and distribute. It is the **canonical, verifiable snapshot** of a compiled knowledge state: what was built, from which inputs, under which policies, with what validation evidence.

The Exchange is designed to be:
- **Deterministic**: same inputs + same policies → same identity
- **Verifiable**: integrity and signatures can be checked offline (fail-closed when declared)
- **Portable**: can be moved across environments and still validate the same way

## What it’s for

Use the Exchange when you need:
- A **stable reference** to a specific compiled knowledge state
- A **package you can archive** and later validate
- A unit of distribution for stores and activation pipelines
- A canonical anchor for associated runtime packs and query capabilities

## What it contains (conceptually)

An Exchange typically includes:
- **Identity & integrity metadata** (content-addressed ID + hashes)
- **Build metadata** (compiler identity, deterministic claim, configuration fingerprint)
- **Input references** (snapshots, recipes/subsets, prior exchanges if incremental)
- **Policies** that affect outputs (the “portable comparability surface”)
- **Profile declarations** (what checks/projections were applied)
- **Evidence index** linking to **validation reports**
- **Payload files** (compiled representations, indexes, manifests, etc.)
- **Signatures** (who attests to this Exchange)

(Exact field names live in the schema/spec; the wiki stays conceptual.)

## Who produces and who consumes it

**Produced by**
- The compiler/build pipeline (and optional validation tooling)

**Consumed by**
- Distribution stores (publish/activate)
- Verifiers and policy gates (CI, compliance)
- Runtime pack builders
- Query services that need a canonical anchor

## Lifecycle placement

Exchange sits in the standard flow:

1. Build (compile)
2. Validate (profiles + evidence)
3. **Publish Exchange**
4. Build/publish Runtime Pack(s)
5. Activate in a store
6. Query

## Common variants

- **Full exchange**: complete domain snapshot
- **Subset exchange**: curated subset (via a recipe)
- **Shard exchange**: a domain-scoped exchange intended for federation/composition

## Related pages

- [[Artifact-Runtime-Pack]]
- [[Artifact-Validation-Report]]
- [[Workflow-Build-and-Validate]]
- [[Workflow-Publish-and-Distribute]]

## Tech details (links)

- Core/spec: `01-core-spec/kristal-v3-core-spec.md`
- Schema: `02-schemas/exchange-manifest.schema.json`
- Example: `10-examples/exchange.example.json`