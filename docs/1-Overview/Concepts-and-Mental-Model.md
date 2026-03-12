# Concepts & Mental Model

This page explains Kristal in “system terms”: what the pieces are, how they fit, and what guarantees you should expect.

---

## The mental model in one sentence

**Kristal turns messy upstream data into a portable, verifiable, deterministic artifact that you can safely distribute and query.**

---

## The core loop

1. **Ingest** upstream sources (snapshots, dumps, feeds)
2. **Compile** them into normalized internal representations
3. **Validate** structure and invariants (schemas + profiles)
4. **Package** outputs into distributable artifacts
5. **Distribute** and **activate** into a runtime environment
6. **Query** with declared capabilities
7. **Evolve** safely (new versions, patches, rollback)

---

## Key concepts

### Kristal
A “Kristal” is the unit you build and operate. Practically, it is represented by one or more artifacts (Exchange, Runtime Pack, etc.) that have stable identity and verification signals.

### Exchange vs Runtime Pack
- **Exchange**: the distributable “bundle” that represents the knowledge payload and its metadata.
- **Runtime Pack**: an optimized package for execution and query (indexes, layouts, policies), derived from Exchange and meant for fast activation.

### Validation
Validation is how consumers gain confidence:
- schema validity (shape of files / manifests)
- profile conformance (rules and higher-level invariants)
- integrity checks (hashes/signatures when present)

### Identity
Identity answers: “What exactly is this artifact?”
- In Kristal, identity is tied to the content (and the rules used to canonicalize/hash it).
- Identity enables dedup, caching, reproducibility checks, and safe rollbacks.

See: [Identity & Determinism](Identity-and-Determinism)

### Determinism
Determinism answers: “If we rebuild from the same inputs under the same policies, do we get the same thing?”
- Determinism is recorded explicitly (inputs, build config, portable policies).
- Consumers can demand determinism as a gate.

### Trust & Authority
Trust answers: “Who is allowed to assert what?”
- Signatures attest integrity and/or authority.
- Trust roots and policies decide which signatures are accepted for a given scope.

See: [Trust, Authority & Signatures](Trust-Authority-and-Signatures)

---

## Composition (advanced)

### Shards
A **shard** is a scoped Kristal artifact (e.g., “only domain X, only time window Y”) that can be validated and sealed independently.

### Federations
A **federation** composes multiple shards into a single view without rewriting the shards.
- The federation declares how composition is done (policy) and can be published by a curator.
- Verification remains fail-closed: each shard still must verify on its own terms.

See: [Federation & Curation](Workflow-Federation-and-Curation)

---

## What “fail-closed” means (practically)

If an artifact declares integrity/trust signals, a consumer should treat verification failure as a hard stop:
- missing referenced files
- hash mismatches
- invalid signatures (or unacceptable authorities, when policy is present)

---

## How to read the wiki

- Use **Artifacts** when you need to know “what exists” and “who consumes it”.
- Use **Workflows** when you need to know “how teams operate it”.
- Use **Operations** when you need “how to run it safely”.

Next: [Quickstart](Quickstart)