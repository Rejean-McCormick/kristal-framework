# Sharding & Federation Upgrade (Kristal v3)

**Status:** Draft (architecture + contracts)  
**Version:** 3.0 (extension)  
**Theme:** Domain sharding + multi-authority composition + offline verification

---

## 1. Purpose

Kristal v3’s core model (Exchange + Runtime Packs) is content-addressed, deterministic, and offline-verifiable. This upgrade adds:

1) **Domain sharding**: split canonical truth into independent Exchange units (“shards”) scoped by domain/subdomain/time-slice.
2) **Federation**: compose multiple shards—possibly sealed by different authorities—under a single “federation root” without rewriting or re-signing the underlying shards.

This enables:
- smaller offline packages (targeted domains),
- parallel publication by multiple parties,
- policy-based trust selection (e.g., “Authority A for domain X, my own seal for domain Y”).

---

## 2. Summary (high-level)

### 2.1 New primitives

#### A) Kristal Shard
An immutable, content-addressed Exchange artifact representing a scoped portion of the world model (domain/subdomain/time slice).

#### B) Federated Kristal
A **federation manifest** that references multiple shards and declares deterministic composition rules.

**Key property:** modifying one shard produces a new shard identity and a new federation root; other shards remain intact and verifiable.

---

## 3. Terminology

- **Shard**: a scoped Exchange artifact.
- **Federation Root**: a manifest that composes shards into a usable “whole”.
- **Authority**: the signer (or signer set) that seals a shard or federation root.
- **Seal**: the integrity material binding an authority identity to an artifact identity (hash + signatures).
- **Authority Registry**: pinned, versioned trust policy data mapping scopes to allowed authorities and required profiles.

---

## 4. Artifacts introduced by this upgrade

This upgrade introduces three additional artifact types (in addition to the existing v3 artifacts):

1) **Exchange Shard Manifest** (sidecar)
2) **Exchange Federation Manifest** (root)
3) **Authority Registry** (policy data; pinned/versioned)

Recommended canonical schema locations:
- `02-schemas/exchange-shard-manifest.schema.json`
- `02-schemas/exchange-federation-manifest.schema.json`
- `02-schemas/authority-registry.schema.json`

Recommended examples:
- `10-examples/exchange-shard-manifest.example.json`
- `10-examples/exchange-federation-manifest.example.json`
- `10-examples/authority-registry.example.json`

---

## 5. Shard model

### 5.1 Shard scope
A shard MUST declare a scope that is deterministic and comparable, for example:
- `domain` (e.g., "education", "heritage")
- optional `subdomain`
- optional `time_slice` (e.g., year/month window, snapshot range)
- optional `tenant_id` / `visibility` if used by deployment policy (not required by core)

### 5.2 Shard identity
A shard is content-addressed as a normal Exchange artifact:
- the Exchange payload has its own content-addressed identity (v3 rules)
- signatures are overlays and are excluded from the hash target (v3 rules)

### 5.3 Shard sealing and validation
A shard MUST be accompanied by:
- integrity declaration (hash + signatures/attestations), and
- validation references (one or more validation reports), so consumers can enforce acceptance policies offline.

---

## 6. Federation model

### 6.1 Federation Root identity
A federation root is content-addressed from its own hashed material (manifest content excluding signature overlays).

### 6.2 Composition is deterministic
A Federation Manifest MUST define a deterministic composition policy, including:
- shard precedence rules when multiple shards cover overlapping scope,
- deterministic conflict resolution (e.g., authority precedence, latest time slice, explicit deny/allow lists),
- stable ordering requirements for shard lists and policy lists.

### 6.3 Federation does not rewrite shards
A federation root MUST NOT mutate shard bytes, shard identities, or shard seals. It only references them and declares how to compose them.

---

## 7. Authority Registry (trust policy data)

### 7.1 Role
Authority Registry defines:
- trust roots and authority identifiers,
- which authorities are allowed for which scopes,
- optional profile requirements (e.g., “must include validation profile X”),
- revocation / deprecation / blocking sets (offline-friendly).

### 7.2 Pinned & versioned
Authority Registry MUST be treated as pinned/versioned policy data:
- consumers must be able to verify with no network access,
- updates are explicit (new registry version / new signed registry payload),
- deployments decide how registry updates propagate (channels, sync, etc.).

---

## 8. Offline verification semantics (fail-closed)

A consumer verifying a federated kristal MUST be able to do so offline given:
- federation manifest bytes,
- referenced shard bytes,
- referenced validation report bytes (or their sealed identities),
- an Authority Registry (pinned/versioned),
- trust roots needed to verify declared signatures.

### 8.1 Verify a shard
A consumer MUST:
1. Validate shard manifest schema
2. Validate shard Exchange identity and content_hash consistency (per v3 rules)
3. Verify declared signatures/attestations (fail-closed if required)
4. Verify referenced validation reports (schema-valid, signature-valid if declared)
5. Check authority acceptance for the shard’s scope using Authority Registry
6. Accept or reject shard deterministically

### 8.2 Verify a federation
A consumer MUST:
1. Validate federation manifest schema
2. Verify federation signature overlays if declared (fail-closed if required)
3. For each referenced shard: run shard verification (8.1)
4. Apply deterministic composition_policy
5. If any required shard fails verification, federation acceptance fails (unless policy explicitly allows optional shards and marks them as optional)

---

## 9. Suggested minimal JSON shapes (non-normative)

### 9.1 Exchange Shard Manifest (sketch)
```json
{
  "schema_version": "3.0",
  "shard_id": "sha256:...",
  "scope": { "domain": "heritage", "subdomain": "unesco", "time_slice": "2026-01" },
  "exchange_ref": { "kristal_id": "sha256:..." },
  "canonicalization_profile": "kristal.v3:jcs-rfc8785",
  "canonicalization_version": "1",
  "validation_refs": [{ "report_id": "sha256:...", "role": "core" }],
  "integrity": { "content_hash": { "alg": "sha256", "value": "..." } },
  "authority": { "authority_id": "authority:example", "kid": "..." },
  "signatures": [ /* standard v3 signature objects */ ]
}
````

### 9.2 Federation Manifest (sketch)

```json
{
  "schema_version": "3.0",
  "federation_id": "sha256:...",
  "canonicalization_profile": "kristal.v3:jcs-rfc8785",
  "canonicalization_version": "1",
  "shards": [
    { "shard_id": "sha256:...", "ref": "uri-or-path", "seal_ref": "..." }
  ],
  "composition_policy": { "mode": "deterministic", "precedence": ["recognized", "personal"] },
  "publisher_seal": { "authority_id": "authority:publisher", "kid": "..." },
  "signatures": [ /* standard v3 signature objects */ ]
}
```

### 9.3 Authority Registry (sketch)

```json
{
  "schema_version": "3.0",
  "registry_id": "sha256:...",
  "trust_roots": {
    "recognized": [{ "authority_id": "authority:unesco", "kids": ["..."] }],
    "personal": [{ "authority_id": "authority:me", "kids": ["..."] }]
  },
  "rules": [
    { "scope": { "domain": "heritage" }, "allow": ["authority:unesco"], "require_profiles": ["profile-validation-shacl@1"] },
    { "scope": { "domain": "notes" }, "allow": ["authority:me"], "require_profiles": [] }
  ],
  "pin_sets": {
    "active": ["authority:unesco", "authority:me"],
    "deprecated": [],
    "blocked": []
  },
  "revocation_list_ref": { "ref": "revocations.json", "sha256": "..." },
  "signatures": [ /* standard v3 signature objects */ ]
}
```

---

## 10. Compatibility and migration

* This upgrade is additive: existing v3 Exchanges and Runtime Packs remain valid.
* Shards are simply scoped Exchanges; federation is a new composition layer.
* Deployments can introduce federation gradually:

  * publish shards first,
  * then publish federation roots referencing them,
  * then add authority registry policy enforcement.

---

## 11. Conformance (minimum)

An implementation supporting sharding/federation MUST:

* remain offline-correct and enforce fail-closed verification where integrity is declared,
* compute and verify identities per v3 canonicalization/hashing rules,
* implement deterministic composition policies,
* implement authority-policy checks using a pinned/versioned authority registry.

---

## 12. Open questions (to finalize)

* Do we standardize a single canonical `scope` schema (domain/subdomain/time) or allow multiple scope kinds with a required discriminator?
* Do we require federation roots to declare whether shards are optional vs mandatory?
* How are overlapping statements handled when two shards assert different values (authority precedence vs explicit deny/allow)?
* Do we standardize a revocation artifact format, or keep revocation as deployment-specific policy data?

