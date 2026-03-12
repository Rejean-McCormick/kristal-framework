# Workflow: Activate, Rollback & Downgrade

This workflow explains how Kristal deployments **activate** a published build (Exchange + Runtime Pack(s)) into a store, and how they safely **rollback** or **downgrade** when something goes wrong.

The goal is operational safety:
- Activation is **atomic** (either the new version is live, or it isn’t)
- Rollback is **fast** and **deterministic**
- Downgrade rules are **explicit** (avoid unsafe mixing of incompatible artifacts)

---

## 1) Inputs

You typically have:

- A published **Exchange**
- One or more published **Runtime Packs**
- **Validation evidence** (reports) and (optionally) **signatures**
- A target **store/environment** (tenant, channel, region)

---

## 2) Activation steps (high level)

### Step A — Resolve what to activate
Pick the exact versions:
- Exchange ID
- Runtime Pack ID(s)
- (Optional) Federation root ID, if you deploy a federated view

### Step B — Verify (fail-closed if declared)
Before activation, verify what your policy requires:
- Integrity (hashes match declared values)
- Signatures (if required by policy)
- Validation gates (profiles required for that environment)

If something is declared and doesn’t verify → **do not activate**.

### Step C — Stage (download / unpack / precompute)
Prepare the new version without impacting the currently active one:
- fetch artifacts
- unpack runtime pack(s)
- build any local indexes/caches required by the query layer

### Step D — Switch atomically
Update the store’s “active pointer” in one step:
- from current version → new version
- write a durable activation record (timestamp, operator, reason, correlation id)

### Step E — Post-activation checks
Run quick checks to catch obvious issues:
- query smoke tests
- performance sanity checks (latency, memory)
- consistency checks (expected dataset size / row counts, etc.)

---

## 3) Rollback (same generation)

Rollback means: **return to the previously active version** (same major compatibility).

### When to rollback
- activation smoke tests fail
- query layer unstable
- unexpected correctness regressions

### How to rollback
- revert the active pointer to the previous activation record
- keep the failed candidate staged for debugging (optional)
- record rollback reason and correlation id

Rollback should be fast because the previous version was already known-good.

---

## 4) Downgrade (cross generation)

Downgrade means: moving back to an older version that may have:
- older policies
- older runtime pack format
- older query contract or integration expectations

### Downgrade safety rules
- Only downgrade if your compatibility policy allows it
- Avoid mixing:
  - new Exchange with old Runtime Pack (or vice versa)
  - new federation root with old shard formats

### Recommended practice
- Treat downgrade as a controlled, policy-gated operation
- Keep a small set of “approved downgrade targets” (blessed releases)
- Run the same verification + smoke test gates as activation

---

## 5) Operational checklist

Before activation:
- [ ] artifacts resolved (exact IDs)
- [ ] integrity verified (hashes)
- [ ] signatures verified (if required)
- [ ] required profiles present/passed
- [ ] staged and ready

After activation:
- [ ] smoke test queries pass
- [ ] key metrics stable (latency/error rate)
- [ ] activation record written

On rollback:
- [ ] pointer reverted atomically
- [ ] rollback recorded (reason + correlation id)
- [ ] follow-up ticket created

---

## Related pages

- [[Workflow-Publish-and-Distribute]]
- [[Operations-Release-Strategy]]
- [[Operations-Observability-and-Troubleshooting]]
- [[Operations-Compatibility]]

## Tech details (links)

- Integration: `06-integration/konnaxion-distribution-contract.md`
- Security/Ops: `07-security/downgrade-rollback-policy.md`
- Ops: `08-ops/release-strategies.md`