# Artifact: Authority Registry

The **Authority Registry** is the pinned policy data a consumer uses to decide **which authorities are acceptable** for a given scope (domain/subdomain/tenant/time slice), and how to verify them.

It’s the piece that makes federation “safe by default” in multi-publisher systems.

---

## What it is (in one sentence)

A versioned, content-addressed policy artifact that lists:
- **trust roots** (who you’re willing to trust), and
- **rules** (where that trust applies).

---

## When you need it

You typically need an Authority Registry when:
- you aggregate shards from multiple publishers/organizations
- you accept shards only for certain scopes (e.g., “UNESCO can publish `heritage/unesco`”)
- you want offline verification without dynamic policy calls

If you have a single trusted publisher and no federation, you may not need it.

---

## What it contains (high level)

### 1) Trust roots
A set of identifiers or key material you treat as anchors, usually separated by categories such as:
- **personal** (local/individual/tenant-scoped)
- **recognized** (organizations, curated lists)
- (optional) active / deprecated / blocked sets for rotation and incident response

### 2) Scope rules
Rules that map a scope to requirements such as:
- which authority classes are allowed (recognized only vs personal allowed)
- which authorities (or key IDs) are allowed for that scope
- which validation profiles must be present (if you enforce profile gating)

### 3) Revocation / rotation hints (optional)
Pointers to revocation lists or “blocked keys” for quick fail-closed decisions.

---

## How it is used

A consumer verifying a shard or federation typically:
1) loads the Authority Registry referenced by the federation (or by local config)
2) finds the relevant rule for the shard’s scope
3) checks the shard’s authority/seal against the allowed roots/keys
4) enforces any declared validation/profile requirements
5) fails closed if integrity is declared and verification does not pass

---

## How it relates to other artifacts

- **Shard Manifest**: declares *who* sealed the shard and what scope it covers  
- **Federation Manifest**: may reference the Authority Registry used to verify shards  
- **Validation Reports**: provide evidence that required checks ran and passed

---

## Operational notes

### Keep it pinned and versioned
Treat the registry as content-addressed policy data:
- publish new versions explicitly
- keep previous versions for auditability and rollback

### Rotate safely
Use active/deprecated/blocked sets so you can:
- roll forward to new keys
- invalidate compromised keys quickly

---

## Tech details (links)
- Schema: (link to `02-schemas/authority-registry.schema.json`)
- Example: (link to `10-examples/authority-registry.example.json`)
- Related: Trust & signatures docs (link to core trust/signatures spec)