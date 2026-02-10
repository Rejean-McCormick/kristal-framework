# Sharding & Federation Upgrade (Kristal)

**Path (suggested):** `docs/02-architecture/03-sharding-and-federation.md`  
**Status:** Normative (architecture + contracts)  
**Goal:** Add **domain sharding** and **multi-authority validation** so anyone can publish a Kristal under their own seal, while composing with shards sealed by recognized authorities (e.g., UNESCO).

> Note: une partie des dumps de docs précédents a expiré côté workspace. Ce document est rédigé pour être compatible avec le modèle discuté ici; si tu veux un alignement mot-à-mot avec les fichiers existants, ré-uploade le dump.

---

## 1) Summary

This upgrade introduces two new primitives:

1) **Kristal Shards (per-domain Exchanges)**  
   Canon truth is split into **independent Exchange units** (“shards”) at a chosen granularity (domain/subdomain/time-slice). Each shard is immutable and sealed (hash + signature).

2) **Federated Kristal (root manifest that composes shards)**  
   A higher-level “Kristal” becomes a **federation manifest** that references multiple shards—potentially issued/validated by different authorities—without re-signing or rewriting them. Consumers can apply policies like “accept UNESCO for domain X” or “accept my own seal for domain Y”.

Key property: **modifying one shard produces a new shard ID and a new federation root**, but **other shards remain intact and verifiable**.

---

## 2) Definitions

### 2.1 Shard
A **Shard** is a Kristal Exchange artifact scoped to a subset of canon:
- **scope**: domain / subdomain / dataset / time-slice
- **truth**: immutable statements + IDs
- **seal**: integrity envelope (hashes + signature(s))
- **validation**: declared validation profiles + signed validation report(s)

### 2.2 Authority
An **Authority** is any entity that can sign:
- a Validation Report, and/or
- an Exchange Manifest, and/or
- a Federation Manifest.

An authority is identified by `authority_id` and anchored by a **trust root** (public key / certificate / DID / keyset) scoped per tenant or channel.

> Design rule: **any authenticated user may be an authority** for their own Kristals (personal trust root), while “recognized” authorities (UNESCO, lab, university) are additional trust roots.

### 2.3 Federation Root (Federated Kristal)
A **Federation Root** is a manifest that:
- lists shard references,
- records their authority seals and validation metadata,
- optionally adds composition rules and precedence,
- is itself sealed (signed) by the “publisher/curator” (could be a solo researcher).

Federation root **does not mutate** shards; it only composes them.

---

## 3) Goals / Non-goals

### 3.1 Goals
- Allow **any authenticated user** to:
  - run validation,
  - compile a shard,
  - seal/sign it,
  - publish it to a channel they control.
- Allow composition:
  - a solo researcher can publish a federated Kristal referencing **UNESCO shards** + **their own shards**.
- Preserve invariants:
  - immutability,
  - fail-closed verification under declared trust roots,
  - deterministic builds for a fixed input/policy set,
  - no-new-facts downstream (render).

### 3.2 Non-goals
- “Editing an existing sealed shard in place” (forbidden). Updates are new shards.
- Treating a federation root signature as “upgrading” UNESCO validity. UNESCO validity remains attached to UNESCO signatures only.

---

## 4) Granularity: “Shard per domain” — what does it mean?

### 4.1 Recommended default granularity
Start with **domain-level shards**:
- one shard per domain of knowledge (e.g., `education`, `heritage`, `climate`, `lab-notes`, `course-material`)
- optionally per tenant (`tenant_id`) and visibility class (`public|restricted|private`)

### 4.2 When to split further
Split into **subdomain/time-slice shards** when:
- the update cadence differs (e.g., weekly lab results vs yearly UNESCO releases),
- the dataset is large (performance/distribution),
- access control differs (private research + public facts).

### 4.3 Anti-pattern: over-sharding
Too fine (per-claim shard) increases:
- manifest overhead,
- signature operations,
- distribution fragmentation,
- rendering/query complexity.

Rule of thumb: a shard should be “small enough to evolve independently” but “large enough to be operationally manageable”.

---

## 5) Contract changes (what gets added)

### 5.1 New artifact: Shard Manifest
**Suggested schema file:** `docs/04-artifacts-contracts/schemas/exchange-shard-manifest.schema.json` (TBD)

Minimum fields:
- `shard_id` (content-addressed)
- `scope` (domain/subdomain/time window)
- `producer` (compiler identity)
- `inputs` (snapshots + policies + blueprint hash)
- `validation_refs[]` (one or more validation reports)
- `integrity` (hashes + signatures)
- `authority` (who sealed it)

### 5.2 New artifact: Federation Manifest
**Suggested schema file:** `docs/04-artifacts-contracts/schemas/exchange-federation-manifest.schema.json` (TBD)

Minimum fields:
- `federation_id` (content-addressed)
- `shards[]` (refs + their seals)
- `composition_policy` (precedence / conflict handling)
- `publisher_seal` (signature of the curator)
- `trust_requirements` (optional policy hints: “domain X must be UNESCO-signed”)

### 5.3 Authority registry (policy surface)
A tenant/channel must declare a set of accepted trust roots:
- `trust_roots.personal[]` (any user’s personal keys if allowed)
- `trust_roots.recognized[]` (UNESCO, university, lab keys)
- `rules` mapping:
  - which authorities are accepted for which domains,
  - which validation profiles are required for publication/activation/render.

This registry is policy data (pinned/versioned) and participates in determinism.

---

## 6) Pipeline changes (Orgo / Kristal) — high-level

### 6.1 Old model (simplified)
`Validate → Compile Exchange → Compile Pack → Publish`

### 6.2 New model (sharded)
For each shard scope:
1) Ingest inputs (snapshots)
2) Extract Claim-IR
3) Resolve → Resolved Claim-IR
4) Validate → Validation Report(s)
5) Compile → **Exchange Shard**
6) Compile → Pack (optional per shard)
7) Publish shard pack(s) (optional)

Then:
8) Compose → **Federation Root** (manifest only)
9) Publish federation root (and optional “federation pack index”)

### 6.3 Who can validate?
**Anyone authenticated can validate** in the following sense:
- anyone can run validators and produce a Validation Report
- they can sign it with their own trust root
- they can compile and seal shards under their authority

But: what becomes “accepted” depends on the **consumer policy** (tenant/channel):
- a personal channel may accept any personal authority,
- a “UNESCO-official” channel accepts only UNESCO trust roots.

This keeps “open validation” compatible with “recognized authority”.

---

## 7) Verification semantics (fail-closed, but plural)

### 7.1 Verifying a shard
A shard is accepted if:
1) its integrity envelope verifies (hash/signature),
2) its validation reports verify (signatures + schema),
3) its authority is allowed by the current policy for that scope.

Fail-closed: if policy says “UNESCO required” and shard is not UNESCO-signed → reject.

### 7.2 Verifying a federation root
A federation is accepted if:
1) the federation manifest verifies (integrity),
2) every referenced shard verifies under policy,
3) composition policy is valid and deterministic.

Important: federation root signature **does not overwrite** shard authority. It only attests: “I publish this composition”.

---

## 8) Composition rules (how shards overlay without lying)

### 8.1 Default composition (safe)
- Each domain scope is disjoint (no overlap in responsibility).
- Queries/render route to exactly one shard per domain.

### 8.2 Allowed overlap (advanced)
If shards overlap (same subject/predicate), federation must declare:
- precedence (which shard wins),
- or “multi-source explicit ambiguity” (preserve both as alternatives).

No silent collapse: overlap without declared rule is invalid under strict policies.

### 8.3 UNESCO example
- `heritage` shard: UNESCO-signed
- `lab-notes` shard: solo researcher-signed
- federation root: researcher-signed
Policy:
- For `heritage`, require UNESCO signature
- For `lab-notes`, allow researcher signature
Rendering:
- outputs that use heritage facts trace to UNESCO shard IDs
- lab notes trace to researcher shard IDs

---

## 9) Rendering / Trace Map impact

Render remains downstream and must preserve “no new facts”.
With sharding:
- trace supports now point to `(shard_id, record_id)` or `(shard_id, claim_id)`
- federation root can be included as context, but supports must still point to the shard where the statement lives.

If the output mixes facts from multiple authorities, trace_map MUST preserve authority provenance per assertion.

---

## 10) Minimal JSON examples

### 10.1 Shard reference inside a federation
```json
{
  "shard_ref": {
    "ref": "kristal:shard:sha256:abc...",
    "scope": { "domain": "heritage", "version": "2026.02" },
    "authority": { "authority_id": "UNESCO", "key_id": "unesco-root-1" },
    "integrity": { "hash": "sha256:...", "sig": "..." },
    "validation_refs": [
      { "ref": "valrep:sha256:...", "authority_id": "UNESCO" }
    ]
  }
}
