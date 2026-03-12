# Operations: Compatibility (v3 / v3.1)

This page explains how to operate Kristal across versions without breaking consumers, and how to plan upgrades safely.

It focuses on **practical compatibility**: what can change, what must not, and how to roll forward/back.

---

## Compatibility goals

A compatibility strategy should ensure:
- consumers can reject incompatible artifacts early (fail closed)
- publishers can roll out changes gradually
- rollback/downgrade remains safe and deterministic

---

## What should remain stable

Across minor versions, you generally want to keep stable:
- artifact identity expectations (IDs change when meaningful content changes)
- integrity semantics (hash/signature verification)
- query contract expectations (capabilities negotiated, pagination rules)

---

## What may change (with coordination)

Common safe evolution areas:
- new optional fields in manifests (backward compatible)
- additional profiles and validations
- new runtime pack policies (if declared and gated)
- new query capabilities (advertised via capabilities)

---

## What is a breaking change

Treat these as breaking unless explicitly version-gated:
- removing required fields or changing their meaning
- changing canonicalization or hashing behavior
- changing signature verification rules
- changing composition policy behavior for federation

Breaking changes should be paired with:
- explicit version bumps
- compatibility notes
- migration guidance
- dual-publish or bridging period

---

## Operating with both v3 and v3.1

A practical approach:

### 1) Dual publishing (recommended)
- publish v3 artifacts and v3.1 artifacts side-by-side
- keep both discoverable
- allow consumers to opt into v3.1

### 2) Capability gating
- consumers declare supported versions/capabilities
- publishers ensure artifacts are produced accordingly

### 3) Explicit activation control
- activation points to a specific runtime pack ID
- activation is atomic
- rollback is always possible to last known-good ID

---

## Upgrade playbook (high level)

1) **Prepare**
- publish the new version artifacts without activating
- validate and run smoke queries
- confirm monitoring dashboards are ready

2) **Canary**
- activate for a small segment (tenant, region, service)
- observe error rates, latency, correctness

3) **Rollout**
- expand activation gradually
- keep old artifacts available

4) **Finalize**
- document what changed
- keep a rollback window
- deprecate old version only after confidence

---

## Rollback and downgrade

Rollback should be:
- **fast** (single pointer flip)
- **safe** (old pack still verifies)
- **auditable** (who changed what, when)

Downgrade is similar but assumes the newer artifacts may remain published:
- the activation target is what matters
- consumers must still fail closed on incompatible inputs

---

## Compatibility checklist

- [ ] Are new fields additive and optional?
- [ ] Are canonicalization and hashing unchanged?
- [ ] Are signatures verifiable with pinned trust roots?
- [ ] Do query capabilities advertise changes?
- [ ] Can you roll back without rebuilding?

---

## Related pages
- [[Workflow-Activate-Rollback-Downgrade]]
- [[Operations-Release-Strategy]]
- [[Operations-Observability-and-Troubleshooting]]
- [[Query-Pagination-and-Capabilities]]