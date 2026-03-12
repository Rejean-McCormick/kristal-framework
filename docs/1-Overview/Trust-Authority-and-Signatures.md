# Trust, Authority & Signatures

This page explains **who is allowed to publish what**, and **how consumers decide what to accept**—without diving into cryptographic implementation details.

## Why this exists
Kristal artifacts are meant to be:
- **Verifiable** (you can check integrity and provenance)
- **Deterministic** (the same inputs produce the same identity)
- **Offline-correct** (verification doesn’t require network calls)

Trust/authority makes it possible to:
- Accept artifacts from the right publisher(s)
- Reject tampered or unapproved artifacts
- Federate multiple sources while keeping clear precedence rules

## Three distinct ideas (don’t mix them)
### 1) Integrity
“Was this artifact changed?”
- Usually: a hash + signature(s) over a defined target

### 2) Authority
“Is this publisher allowed to publish for this scope?”
- Example scopes: `domain`, `subdomain`, time windows, tenant environments
- Authority is evaluated against a pinned policy/registry when used

### 3) Trust Policy
“What do we require before we accept something?”
- Which keys/issuers are valid
- Which scopes they’re valid for
- Which validations are mandatory

## What is signed (conceptually)
For most artifacts:
- There is a **stable, deterministic “signing target”**
- Volatile fields (timestamps, the signatures themselves, transport metadata) are not part of that target
- Consumers verify signatures against the declared target and fail closed when configured to do so

## Fail-closed semantics (high level)
If an artifact declares integrity material (hash/signatures):
- **Verification must fail** if the material is missing or invalid
- Consumers should treat “declared but unverifiable” as unsafe

If an artifact does **not** declare signatures:
- It may still be usable in low-trust environments, but cannot claim the same assurance

## Roles you’ll see in practice
- **Producer / Compiler**: generates artifacts
- **Publisher**: signs and distributes a curated artifact
- **Curator** (federation): composes shards into a federation root and signs that composition
- **Consumer**: verifies, activates, queries

## Shards, federations, and authority
When sharding/federation is used:
- A **Shard** represents an authoritative slice of knowledge (for a scope)
- A **Federation Root** references multiple shards and declares composition rules
- An **Authority Registry** (when used) defines which authorities are accepted for which scopes
- Consumers verify:
  1) shard integrity
  2) shard authority (scope → allowed authority)
  3) federation composition integrity

## What to do if you’re implementing
- Start with **integrity verification** (hash + signature verification)
- Add **authority checks** only when you need multi-publisher governance
- Keep policy data **pinned/versioned** so verification is deterministic

## Common pitfalls
- Confusing “publisher” (who distributes) with “authority” (who is allowed for a scope)
- Treating timestamps as part of identity (breaks determinism)
- Allowing “best effort verification” (should be fail-closed when declared)

## Next pages
- Artifacts → Authority Registry
- Artifacts → Shard Manifest
- Artifacts → Federation Manifest
- Workflows → Activate, Rollback & Downgrade