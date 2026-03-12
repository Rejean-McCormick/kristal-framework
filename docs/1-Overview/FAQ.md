# FAQ

## What is a Kristal, in one sentence?
A Kristal is a **verifiable, deterministic, offline-executable knowledge bundle** you can publish, activate, and query reliably.

## Is Kristal a database?
Not exactly. Kristal produces **portable artifacts** (Exchange + Runtime Pack(s)) that can be queried like a local database, but it’s primarily a **build/verify/distribute framework**.

## Why do “determinism” and “content-addressed IDs” matter?
They make results **reproducible** and **auditable**:
- the same inputs/policies yield the same identity
- you can verify that what you received is exactly what was built

## What’s the difference between Exchange and Runtime Pack?
- **Exchange**: canonical published bundle (what was compiled, under what policies, with what evidence)
- **Runtime Pack**: optimized local format for fast queries

## What are “profiles”?
Profiles are named configurations for:
- validation rules (shape, integrity, domain checks)
- projections/exports
- query behaviors (pagination, capabilities)

They let deployments standardize behavior without rewriting the core.

## What is a Validation Report?
A Validation Report is evidence that declared checks ran and passed (or failed). Deployments can gate activation on required reports.

## What does “fail-closed” mean here?
If an artifact **declares** hashes/signatures/evidence, and verification fails or is missing, the consumer should treat it as **invalid** (not “best effort”).

## What are Shards and why use them?
A **shard** is a domain-scoped published unit that can be updated independently. Use shards when:
- you have multiple publishers/domains
- you want independent update cadence
- you need explicit authority boundaries

## What is a Federation Manifest?
It’s the “composition root” that declares:
- which shards are included
- deterministic merge rules
- optional trust requirements and references

It gives clients a single entrypoint to a multi-shard world.

## Do I need an Authority Registry?
Not always. If your deployment needs explicit trust policy (“which authorities are recognized for this scope”), an Authority Registry makes that policy **pinned and auditable**.

## Can I rollback or downgrade safely?
Yes, if you treat activation as atomic and follow explicit compatibility rules:
- rollback returns to the previously active version
- downgrade is policy-gated and avoids mixing incompatible artifacts

## Where are the technical details?
This wiki focuses on overview and usage. For exact formats/schemas/specs, use the referenced docs:
- Core/specs: `kristal-docs-v3/01-core-spec/...`
- Schemas: `kristal-docs-v3/02-schemas/...`
- Examples: `kristal-docs-v3/10-examples/...`