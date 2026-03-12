# Workflow: Build & Validate

This workflow describes how Kristal artifacts are produced and proven correct before distribution. It’s written for operators and integrators—not as a compiler spec.

## Goal
Produce a set of artifacts that are:
- **Deterministic** (repeatable outputs under recorded policies)
- **Verifiable** (integrity + validation evidence)
- **Ready to publish** (safe for consumers to activate)

## Inputs
Typical inputs include:
- Source snapshot (dataset dump, event stream snapshot, curated export)
- Build configuration (policies that affect output shape)
- Optional subset recipe (if producing a scoped or reduced Kristal)

## Outputs (what you should expect)
At minimum:
- **Exchange** (the publishable unit)

Commonly:
- **Runtime Pack** (optimized for query/render)
- **Validation Report** (proof of checks)
- Optional: **Shard Manifest** (if building a shard)
- Optional: **Authority Registry** reference (if governed publishing is enabled)

## Step-by-step

### 1) Prepare the build
- Select the source snapshot
- Select configuration/policies (ordering, grouping, filters, etc.)
- Decide whether you are producing:
  - a full Exchange, or
  - a scoped subset (recipe-driven), or
  - a shard intended for federation

**Output of this step:** a build plan you can reproduce.

### 2) Build the Exchange
- Run the compiler over the snapshot and config
- Produce the Exchange artifact
- Record build metadata needed for reproducibility

**Key check:** the Exchange identity should be stable for the same inputs and policies.

### 3) Build the Runtime Pack (recommended)
- Generate a runtime representation for query/render
- Record any policies that affect runtime layout/behavior
- Ensure the pack references the Exchange it was built from

**Key check:** the Runtime Pack should be reproducible under the same inputs/policies.

### 4) Validate (fail-closed where declared)
Run the validations required by your environment, typically:
- Structural/schema checks
- Determinism/integrity checks
- Profile validations (domain rules, RDF integrity, etc.)

Produce a **Validation Report** that records:
- which checks ran
- which versions/tools were used
- pass/fail status
- integrity references (hashes) for artifacts being validated

**Key check:** if integrity is declared, missing/mismatched evidence must fail closed.

### 5) Seal / sign (optional but recommended)
If your environment uses signatures:
- Sign the artifact(s) using a stable signing target
- Ensure signatures and timestamps do not change the artifact identity
- If sharding is enabled, ensure the shard authority is clear

### 6) Package for publish
Prepare the publishable bundle:
- Exchange (+ its manifests)
- Runtime Pack
- Validation Report(s)
- Optional shard/federation/authority artifacts

## Acceptance checklist (practical)
Before publishing, confirm:
- You can rebuild and get the same identities (under the same inputs/policies)
- Validation reports exist and show pass for required checks
- Integrity material verifies (hash/signatures)
- Runtime Pack references the correct Exchange
- If authority rules apply, the publisher is authorized for the scope

## Common failure modes
- Non-deterministic ordering (IDs drift between builds)
- Policy metadata missing (can’t reproduce)
- “Best effort” validation (should be fail-closed if declared)
- Runtime pack built from a different Exchange than advertised

## Next pages
- Workflow: Publish & Distribute
- Artifact: Exchange
- Artifact: Runtime Pack
- Artifact: Validation Report
- Workflow: Activate, Rollback & Downgrade