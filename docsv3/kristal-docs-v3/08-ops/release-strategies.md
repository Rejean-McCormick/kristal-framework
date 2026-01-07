# Release Strategies for Kristal Runtime Packs (Blue/Green, Canary)

## Status
Draft (non-normative operational guidance)

## Purpose
Define operational release strategies for distributing **Kristal Runtime Packs** safely (especially to offline or constrained environments) while preserving Kristal v3’s requirements:
- deterministic artifacts,
- fail-closed integrity,
- auditable release records,
- downgrade/rollback safety.

This document is **non-normative**: it does not change Kristal artifact formats or conformance. It provides recommended operational patterns for Orgo (control plane) and Konnaxion (distribution layer).

---

# 1. Terms and goals

## Terms
- **Pack**: a Kristal Runtime Pack artifact (manifest + payloads).
- **Release**: a versioned publication event (Orgo release record referencing pack_id, signatures, channels).
- **Channel**: a distribution target grouping (tenant, region, device cohort, environment).
- **Cohort**: a subset of a channel (e.g., 1% of devices).

## Goals
- Minimize blast radius of a bad pack release (bad data, bad build, bad performance).
- Preserve offline correctness: never run a pack whose integrity fails verification.
- Support fast rollback without creating “downgrade vulnerabilities.”
- Maintain auditable linkage: build → release → distribution → device state.

---

# 2. Release invariants (recommended)

These invariants should hold regardless of rollout strategy:

1. **Immutable pack identity**
   - A `pack_id` identifies immutable content.
   - Never “fix” a pack in place; publish a new pack.

2. **Fail-closed verification**
   - If signatures/hashes are declared, every distribution node and client should verify before activation.

3. **Auditable release records**
   - Every activation should be traceable to a Release record with pack_id, signing key, timestamp, and channel.

4. **Explicit rollback policy**
   - Rollback is allowed only to known-good versions within a defined safe window, respecting minimum-version pins.

---

# 3. Blue/Green releases (recommended default)

## 3.1 Concept
Maintain two production slots per channel:
- **Blue**: current active pack version
- **Green**: new candidate pack version

Devices or nodes only switch to Green once Green has been validated in production-like conditions.

## 3.2 Procedure
1. **Publish Green**
   - Orgo creates a Release referencing the new pack_id.
   - Distribution replicates pack payloads to the Green slot.

2. **Verify at rest**
   - Edge nodes/devices verify:
     - manifest hash,
     - payload hashes,
     - signatures (if present).
   - Verification failure blocks activation.

3. **Warm-up / readiness checks**
   - Run pack-local checks:
     - query smoke tests (basic triple patterns),
     - join-cap behavior,
     - performance sanity (latency thresholds),
     - memory footprint thresholds.
   - Log outcomes with correlation IDs.

4. **Switch traffic**
   - Flip the channel pointer from Blue → Green.
   - Keep Blue retained for a defined rollback window.

5. **Post-switch monitoring**
   - Observe error codes, query latencies, resource usage.
   - If thresholds exceed, trigger rollback.

## 3.3 Advantages
- Simple mental model.
- Predictable rollback.
- Works well for offline package distribution where clients may activate asynchronously.

## 3.4 Pitfalls
- Storage overhead (two slots).
- Must define the activation pointer clearly to avoid split-brain (some devices on Blue, some on Green is expected; the important part is auditability).

---

# 4. Canary releases (recommended for high-risk changes)

## 4.1 Concept
Roll out the new pack to a small cohort first, validate, then expand.

## 4.2 Procedure
1. **Create cohorts**
   - Example: 1% → 10% → 50% → 100%
   - Cohorts should be stable and deterministic (hash(device_id) mod 100).

2. **Canary publish**
   - Orgo creates a Release for cohort 1%.
   - Distribution makes the pack available to the cohort only.

3. **Activation gating**
   - Devices verify integrity and run smoke tests before activation.

4. **Promote**
   - If health metrics are good for a defined observation window, expand to next cohort.

5. **Abort and rollback**
   - If health metrics fail, stop promotion and revert cohorts to the previous pack pointer.

## 4.3 Metrics to watch
- Integrity verification failures
- Query error rates (especially strict join-cap errors)
- Median/P95 query latency for representative workloads
- Memory pressure / OOM events on constrained devices
- Pack download failures or excessive download size complaints

## 4.4 Pitfalls
- Requires cohort assignment stability and a clean mechanism for cohort targeting.
- Offline devices may “skip” cohorts; treat cohort checks as “best effort” and rely on integrity + smoke tests at activation time.

---

# 5. Rollback and downgrade safety (critical)

Rollback is operationally necessary, but uncontrolled downgrade is a security risk.

## 5.1 Recommended rules
- **Rollback pointer**: channel pointer can move back to a prior pack_id only if that pack_id is:
  - still within the retention window,
  - still signed by a trusted key,
  - not below the channel’s **minimum pinned version**.

- **Minimum version pin**
  - Maintain a per-channel `min_allowed_release_version`.
  - Clients MUST NOT activate packs below this minimum (even if cached).

- **Revocation**
  - If a pack is discovered invalid/vulnerable, revoke it:
    - mark as revoked in release metadata,
    - distribute revocation list to clients,
    - bump minimum pin to exclude it.

(See also `07-security/downgrade-rollback-policy.md`.)

---

# 6. Offline and constrained environments (practical notes)

## 6.1 Activation is local and asynchronous
Offline devices may:
- download a pack later than publication,
- activate it later than download.

Therefore:
- integrity verification MUST be done at activation time.
- store the activation decision with an audit marker (pack_id + release_id + timestamp).

## 6.2 Storage constraints
If devices cannot hold Blue+Green simultaneously:
- hold the previous pack’s manifest + signatures at minimum, or
- implement a “checkpoint pack” strategy: keep last known-good pack payloads for rollback where feasible.

## 6.3 Delta updates (optional)
If packs are large:
- use delta distribution at the transport layer,
- but ensure the resulting final payload hashes match the declared pack manifest hashes.

---

# 7. Suggested control-plane data model (informative)

A practical Orgo/Konnaxion release system typically maintains:

- `BuildRecord(build_id, pack_id, kristal_id, hashes, signatures, policies, inputs, compiler_version, config_hash)`
- `Release(release_id, channel_id, pack_id, signing_key_id, published_at, status)`
- `ChannelPointer(channel_id, active_release_id, min_allowed_release_version)`
- `DeviceState(device_id, channel_id, active_pack_id, last_verified_at, verification_status)`

This is illustrative; exact implementation is system-specific.

---

# 8. Recommended “default strategy”

For most deployments:
- Use **Blue/Green** as the default.
- Add **Canary** for high-risk changes:
  - new policies (ordering, row-groups, filters),
  - large schema shifts,
  - new query extensions,
  - new client versions.

---

# 9. Checklist

## Blue/Green
- [ ] Pack published + verified at rest
- [ ] Smoke tests pass
- [ ] Switch pointer
- [ ] Monitor and rollback window defined
- [ ] Minimum version pin updated when needed

## Canary
- [ ] Cohorts stable and targetable
- [ ] Observation metrics defined
- [ ] Promotion steps defined
- [ ] Abort/rollback is one operation
- [ ] Offline activation still verifies integrity + runs smoke tests
