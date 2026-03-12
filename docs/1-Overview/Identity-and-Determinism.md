# Identity and Determinism

Kristal is designed so that **two independent builders** can produce **the same identifiers** and **the same verified artifacts** from the same inputs and policies. This enables offline verification, reliable distribution, and safe rollbacks.

This page explains the *why* and the *what* (not the low-level hashing rules).

---

## What “Identity” means in Kristal

**Identity** is a stable, content-derived handle for an artifact (e.g., an Exchange, a Runtime Pack, a shard, a federation root). In practice:

- If the *meaningful content* is the same, the ID should be the same.
- If the meaningful content changes, the ID must change.

This makes IDs safe for:
- caching and mirroring
- deduplication
- audit trails
- rollback and downgrade workflows

---

## What “Determinism” means

A deterministic build means:

- Given the same inputs and declared policies,
- the pipeline produces the same canonical outputs,
- and therefore the same IDs and integrity evidence.

Kristal treats determinism as a **contract**, not an optimization.

---

## Why this matters

### Offline verification
A consumer should be able to verify an artifact without calling back to a server:
- verify declared hashes
- verify declared signatures
- verify the artifact matches the claimed ID

### Safe distribution
Content-derived IDs reduce ambiguity:
- you can mirror artifacts across stores
- you can detect tampering by comparing IDs/hashes

### Rollback and downgrade
If IDs are stable and builds are deterministic, rollback is straightforward:
- keep the previous known-good artifact available
- activate the previous ID atomically

---

## What must be recorded for determinism

At a high level, determinism depends on four things being explicitly recorded:

1) **Inputs**
- what snapshot(s) were used
- what prior artifacts are referenced (if any)
- what subset/selection rules were applied (if applicable)

2) **Policies**
- ordering / grouping rules
- membership filters / bitmap strategy
- any other “portable” parameters that affect outputs

3) **Tooling identity**
- compiler name/version
- configuration fingerprint (so two builds can be compared)

4) **Canonicalization choices**
- a single canonicalization profile and version for hashed JSON objects
- applied consistently across the build

---

## What is *not* part of identity

To keep identity stable, some things must *not* affect IDs:

- wall-clock timestamps (except where explicitly required for auditing)
- file ordering that has no semantic meaning
- nondeterministic iteration (maps/sets without deterministic ordering)
- deployment metadata (hostnames, region labels, transient runtime details)

These can still be recorded, but they should not change the content-derived identity.

---

## Fail-closed: determinism + integrity

Kristal assumes **fail-closed verification** when integrity is declared:
- if a manifest declares hashes/signatures and they don’t verify, the artifact is rejected
- if required referenced evidence is missing, the artifact is rejected

This is essential for federation and multi-publisher composition.

---

## Where to go for technical details

- Canonicalization and hashing prerequisites: see the technical docs (IDs + canonicalization rules)
- Signature formats and verification semantics: see the trust/signatures docs
- Deterministic build rules and policies: see reproducibility docs

(These links should point to the relevant pages in your docs repo once you wire them into the wiki.)