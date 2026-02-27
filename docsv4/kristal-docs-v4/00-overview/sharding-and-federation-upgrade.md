# Sharding & Federation Upgrade (Kristal v4)

**Status:** Draft (architecture + contracts)  
**Version:** 3.0 (extension)  
**Theme:** Domain sharding + multi-authority composition + offline verification

---

## 1. Purpose

Kristal v4’s core model (Exchange + Runtime Packs) is content-addressed, deterministic, and offline-verifiable. This upgrade adds:

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
- **Revocations**: pinned, versioned revocation policy data (offline-friendly) used during verification.

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

**Recommended additional policy artifact (offline):**
- `02-schemas/revocations.schema.json`
- `10-examples/revocations.example.json`

---

## 5. Shard model

### 5.1 Shard scope (canonical)
A shard MUST declare a scope that is deterministic and comparable:

- `domain` (required; e.g., `"education"`, `"heritage"`)
- `subdomain` (optional)
- `time_window` (optional; implementation-defined label, e.g., `"2026-01"`, `"2020-2025"`)
- deployment policy MAY add `tenant_id`, `env`, or other fields (but MUST NOT change the meaning of the canonical keys above)

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
- deterministic conflict resolution (e.g., authority precedence, latest time window, explicit allow/deny),
- stable ordering requirements for shard lists and rule lists.

### 6.3 Federation does not rewrite shards
A federation root MUST NOT mutate shard bytes, shard identities, or shard seals. It only references them and declares how to compose them.

---

## 7. Authority Registry (trust policy data)

### 7.1 Role
Authority Registry defines:
- trust roots and authority identifiers,
- which authorities are allowed for which scopes,
- optional profile requirements (e.g., “must include validation profile X”),
- pin sets (active/deprecated/blocked),
- an optional reference to revocations policy data.

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
- revocations policy data (if required by the Authority Registry / deployment policy),
- trust roots needed to verify declared signatures.

### 8.1 Verify a shard
A consumer MUST:
1. Validate shard manifest schema
2. Validate shard Exchange identity and content_hash consistency (per v3 rules)
3. Verify declared signatures/attestations (fail-closed if required)
4. Verify referenced validation reports (schema-valid, signature-valid if declared)
5. Check authority acceptance for the shard’s scope using Authority Registry (+ revocations if required)
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
  "artifact_type": "exchange_shard_manifest",
  "created_at": "2026-02-26T18:05:00Z",

  "shard_id": "sha256:...",
  "canonicalization_profile": "kristal.v3:jcs-rfc8785",
  "canonicalization_version": "1",

  "scope": { "domain": "heritage", "subdomain": "unesco", "time_window": "2026-01" },

  "exchange_ref": { "kristal_id": "sha256:..." },

  "validation_refs": [{ "report_id": "sha256:...", "role": "core" }],

  "integrity": { "content_hash": { "alg": "sha256", "value": "..." } },
  "authority": { "authority_id": "authority:example", "key_id": "..." },
  "signatures": [ /* standard v3 signature objects */ ]
}
````

### 9.2 Federation Manifest (sketch)

```json
{
  "schema_version": "3.0",
  "created_at": "2026-02-26T18:10:00Z",

  "federation_id": "sha256:...",
  "canonicalization_profile": "kristal.v3:jcs-rfc8785",
  "canonicalization_version": "1",

  "authority_registry_ref": {
    "registry_id": "sha256:... (or kristal:authority-registry:sha256:...)",
    "ref": "authority/authority-registry.json",
    "content_hash": { "alg": "sha256", "value": "..." }
  },

  "shards": [
    {
      "shard_id": "sha256:...",
      "scope": { "domain": "heritage", "subdomain": "unesco", "time_window": "2026-01" },
      "shard_manifest_ref": "shards/heritage-unesco-2026-01/shard-manifest.json",
      "shard_manifest_hash": { "alg": "sha256", "value": "..." },
      "seals": [ /* optional copied signatures */ ]
    }
  ],

  "composition_policy": { "policy_id": "deterministic-v1", "policy_version": "1", "parameters": {} },

  "publisher": { "authority_id": "authority:publisher", "key_id": "..." },
  "signatures": [ /* standard v3 signature objects */ ],

  "extensions": {}
}
```

### 9.3 Authority Registry (sketch)

```json
{
  "schema_version": "3.0",
  "artifact_type": "authority_registry",
  "created_at": "2026-02-26T18:00:00Z",

  "registry_id": "kristal:authority-registry:sha256:...",
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
  "revocation_list": {
    "ref": "authority/revocations.json",
    "content_hash": { "alg": "sha256", "value": "..." }
  },
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
  * then add authority registry policy enforcement (and revocations if required).

---

## 11. Conformance (minimum)

An implementation supporting sharding/federation MUST:

* remain offline-correct and enforce fail-closed verification where integrity is declared,
* compute and verify identities per v3 canonicalization/hashing rules,
* implement deterministic composition policies,
* implement authority-policy checks using a pinned/versioned authority registry (and revocations if required by policy).

---

## 12. Open questions (to finalize)

* Do federation roots declare shards as optional vs mandatory (schema flag vs policy-driven)?
* How are overlapping statements handled when two shards assert different values (authority precedence vs explicit deny/allow)?
* Do we standardize a small set of named composition policies (vs fully custom `policy_id`)?

