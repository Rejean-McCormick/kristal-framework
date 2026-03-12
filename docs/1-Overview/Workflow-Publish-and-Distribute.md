# Workflow: Publish & Distribute

This workflow covers how a validated Kristal becomes something you can **share and deploy**: packaging, publishing, distribution, and the handoff to activation.

---

## Goal

- Produce a **versioned, distributable release** of a Kristal
- Ensure consumers can **verify** what they received
- Enable **safe rollout** and **safe rollback**

---

## Inputs

Typically:
- A successfully built **Exchange**
- One or more **Validation Reports** (schema/profile checks)
- Optionally a **Runtime Pack** (if distribution is pack-first)

---

## Outputs

- A published artifact set (Exchange and/or Runtime Pack) in a distribution channel
- A clear “what’s current” pointer (index/registry/feed)
- Metadata for operations (release notes, compatibility notes)

---

## Steps (high level)

### 1) Assign a release identity
- Choose a release name/version (human-readable)
- Record lineage (what Exchange/Pack is being released)

### 2) Package for distribution
- Ensure the artifact set is complete (no missing referenced files)
- Include the manifests and any verification material (hashes/signatures)

### 3) Publish
- Upload/push to the chosen channel (repo, registry, object storage, etc.)
- Publish a pointer/index so consumers can discover the release

### 4) Validate the published form (post-publish checks)
- Consumer-style verification (download + verify + dry-run activate)
- Confirm expected capabilities and compatibility notes are correct

### 5) Announce and roll out
- Communicate what changed (new features, scope changes, compatibility)
- Roll out through your standard deployment process

---

## Verification and “fail-closed” behavior

If the published artifacts declare integrity/trust signals:
- consumers should verify them before activation
- failures (missing files, hash mismatch, invalid signatures) should block activation

---

## Distribution patterns

### Exchange-first
- Publish Exchange as the primary deliverable
- Runtime Packs are generated per environment (or later)

### Pack-first
- Publish Runtime Packs as the deployable units
- Exchange remains the source-of-truth and audit trail

### Federated distribution (multi-publisher)
- Multiple shards are published independently
- A curator publishes a federation root that composes them

See: [Federation & Curation](Workflow-Federation-and-Curation)

---

## Operational notes

- Prefer atomic publish operations (avoid partial releases)
- Keep at least one previous “known-good” release available for rollback
- Treat policy changes (ordering/index formats) as release-significant

See: [Release Strategy](Operations-Release-Strategy)

---

## Tech details

This wiki stays high level. For exact artifact formats and distribution contracts, refer to the technical docs in the main repository.