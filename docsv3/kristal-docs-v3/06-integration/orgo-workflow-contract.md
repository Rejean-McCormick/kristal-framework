# Orgo Workflow Contract (Kristal v3 Integration)

## Status
Draft (normative integration contract)

## Purpose
Define the required **workflow orchestration**, **governance**, and **auditability** behavior for Orgo when Orgo acts as the control plane for Kristal v3 production and distribution.

This contract is concerned with:
- enforcing **pipeline ordering** (ingest → Claim-IR → resolution → validation → publish),
- enforcing **deterministic acceptance gates** (“no compile on fail”),
- ensuring every build is **reproducible** (inputs, config, policy selections recorded),
- and ensuring production/distribution actions are **auditable** and tenant-safe.

This document does **not** define the internal Orgo task model beyond what is necessary for Kristal interoperability.

## Normative language
The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are to be interpreted as normative requirements.

---

# 1. Roles and responsibilities

## 1.1 Orgo is the workflow control plane
Orgo MUST:
- create and manage the lifecycle of Kristal build workflows,
- store workflow state, approvals, and audit logs,
- record build inputs/outputs and their content-addressed identifiers,
- trigger distribution and track distribution status,
- surface deterministic errors/warnings to operators.

Orgo MUST NOT:
- mutate Kristal Exchange artifacts directly as a result of feedback signals,
- bypass deterministic validation gates.

## 1.2 Pipeline systems referenced by Orgo
Orgo orchestrates interactions with:
- Extractors (LLMs / classical / hybrid) that output Claim-IR only
- SenTient for resolution (Resolved Claim-IR)
- Validation engine (deterministic)
- Kristal compiler for Exchange and Runtime Pack creation
- Distribution systems (e.g., Konnaxion package distribution)

---

# 2. Workflow stages (mandatory)

A Kristal build workflow MUST follow this stage order:

1. **Ingest**
2. **Extract → Claim-IR**
3. **Resolve (SenTient) → Resolved Claim-IR**
4. **Validate**
5. **Compile → Exchange**
6. **Compile → Runtime Pack**
7. **Publish/Distribute**
8. **Post-publish verification (recommended)**

Orgo MAY split stages into sub-stages, but MUST preserve the ordering constraints and gating semantics in Section 3.

---

# 3. Gating and determinism (mandatory)

## 3.1 “No compile on fail” (hard gate)
If validation fails:
- Orgo MUST mark the workflow as failed at the Validation stage.
- Orgo MUST NOT invoke Exchange or Runtime Pack compilation.
- Orgo MUST surface the validation report (structured codes + messages) to operators.

## 3.2 Deterministic stage outputs
For each stage, Orgo MUST record the content-addressed references to outputs, including:
- Claim-IR batch IDs
- Resolved Claim-IR batch IDs
- Validation report IDs
- Exchange `kristal_id`
- Runtime Pack `pack_id` and payload hashes

## 3.3 Idempotency and replay
Orgo SHOULD support deterministic replay:
- Given the same input snapshots and build configuration, re-running a stage SHOULD yield the same outputs (subject to probabilistic extractors being frozen or replaced with stored Claim-IR).

Orgo MUST ensure that replay does not bypass integrity checks.

---

# 4. Build record requirements (mandatory)

For every Kristal build, Orgo MUST persist a **Build Record** containing:

## 4.1 Identifiers
- `build_id` (unique)
- `tenant_id` (mandatory in multi-tenant deployments)
- `workflow_id` / `case_id` (link back to Orgo case/task)

## 4.2 Inputs (snapshots)
- `input_snapshots[]`: content-addressed references to all inputs used
  - source dataset snapshots
  - Claim-IR snapshots (if extraction is external/frozen)
  - configuration snapshots

## 4.3 Compiler identity and config hash
- `compiler.name`
- `compiler.version`
- `config_hash` (hash of full build config)
- `canonicalization_profile` + `canonicalization_version`

## 4.4 Policy selections (portable policies)
- `policy_selections` as defined in:
  - `03-reproducibility/allowed-runtime-pack-policies.md`

## 4.5 Outputs
- Exchange output:
  - `kristal_id`
  - declared hashes and/or signatures
- Runtime Pack output:
  - `pack_id`
  - payload hashes
  - manifest hash/signature references

## 4.6 Validation summary
- `validation_status` in `{PASS, FAIL}`
- `validation_codes[]` (stable machine codes)
- `validation_report_ref` (content-addressed reference)

---

# 5. Integrity verification requirements (mandatory)

## 5.1 Pre-publish verification
Before publishing/distributing Exchange or Runtime Pack artifacts, Orgo MUST verify:
- declared content hashes match computed hashes
- declared signatures verify (if present)

If verification fails, Orgo MUST:
- fail the workflow,
- prevent publish/distribution,
- record the failure reason.

## 5.2 Fail-closed behavior
Orgo MUST enforce fail-closed behavior for declared integrity material:
- Orgo MUST NOT “publish anyway” when verification fails.
- Orgo MUST treat malformed integrity fields as errors.

---

# 6. Distribution workflow requirements (mandatory)

## 6.1 Versioned release objects
Orgo MUST create a **Release** record per published version containing:
- release ID/version
- Exchange `kristal_id` reference(s)
- Runtime Pack `pack_id` reference(s)
- target channels (tenants, regions, device cohorts)
- publish timestamp
- verification status

## 6.2 Rollout controls (recommended)
Orgo SHOULD support progressive rollout strategies:
- staged rollouts by cohort
- canary or blue/green releases
- automatic rollback on verification failures or high error rates

These rollout mechanics are operational guidance (not schema-level requirements), but the **release record** MUST be auditable.

## 6.3 Downgrade prevention (mandatory where applicable)
If the distribution environment supports version pinning:
- Orgo MUST provide a mechanism to prevent downgrade to a vulnerable/invalid pack version.
- Orgo MUST record pinned minimum versions per channel.

(See `07-security/downgrade-rollback-policy.md`.)

---

# 7. Multi-tenancy and isolation (mandatory)

## 7.1 Tenant scoping
All workflow records MUST be tenant-scoped:
- Orgo MUST prevent cross-tenant visibility of build inputs/outputs unless explicitly configured for shared artifacts.

## 7.2 Keys and signing
Signing keys MAY be tenant-scoped even if `kristal_id` is global content-addressed.
Orgo MUST:
- record which key(s) signed which release,
- enforce that verification uses the correct trust roots for the tenant/channel.

(See `07-security/key-management-and-trust-roots.md`.)

---

# 8. Feedback loop handling (mandatory)

## 8.1 Feedback must not mutate Exchange directly
Orgo MUST represent feedback as:
- new Cases/Tasks that trigger a new build cycle, or
- structured signals that lead to new Claim-IR proposals.

Orgo MUST NOT:
- edit Exchange artifacts in-place due to votes/curation/user edits.

## 8.2 Provenance linkage (recommended)
Orgo SHOULD link:
- feedback signals → cases/tasks → resulting builds/releases,
to make governance auditable.

---

# 9. Observability requirements (recommended)

Orgo SHOULD emit structured logs/events for each stage with correlation identifiers:
- `build_id`
- `tenant_id`
- `kristal_id`
- `pack_id`
- stage name + outcome
- validation error codes
- signature verification result

(See `08-ops/logging-and-correlation-ids.md`.)

---

# 10. Minimal API surface (informative)

Orgo implementations typically expose:
- `CreateBuildWorkflow(input_snapshots, config_ref, policy_selections)`
- `AdvanceStage(build_id, stage, outputs...)`
- `GetBuildRecord(build_id)`
- `CreateRelease(build_id, channels...)`
- `GetReleaseStatus(release_id)`

This section is informative; API shape is implementation-specific.

---

# 11. Conformance checklist (Orgo)

Orgo satisfies this contract if it:
- enforces stage ordering and “no compile on validation failure”,
- records build inputs/config/policies/outputs in auditable Build Records,
- verifies hashes/signatures before publish and fails closed on mismatch,
- creates auditable versioned Release records,
- isolates multi-tenant workflows and enforces correct trust roots,
- ensures feedback triggers new build cycles rather than in-place edits.
