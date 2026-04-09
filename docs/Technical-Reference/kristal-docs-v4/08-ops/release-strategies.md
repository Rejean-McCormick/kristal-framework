# Release Strategies for Kristal Runtime Packs (Blue/Green, Canary)

## Status

Draft (non-normative operational guidance)

## Purpose

Define operational release strategies for distributing **Kristal Runtime Packs** safely (especially to offline or constrained environments) while preserving Kristal v4’s requirements:

* deterministic artifacts,
* fail-closed integrity,
* auditable release records,
* downgrade/rollback safety.

This document is **non-normative**: it does not change Kristal artifact formats or conformance. It provides recommended operational patterns for Orgo (control plane) and Konnaxion (distribution layer). It assumes channels use a signed channel index (e.g., `pack_index.json`) as the primary control surface for rollout/rollback. 

---

# 1. Terms and goals

## Terms

* **Pack**: a Kristal Runtime Pack artifact (manifest + payloads).
* **Pack version**: a monotonic version within a channel used for activation and downgrade prevention (`pack_version`). 
* **Release (Orgo)**: a versioned publication event (Orgo release record referencing `pack_id`, signer, channel scope).
* **Channel**: a distribution target grouping (tenant, environment, region, device cohort).
* **Channel pack index**: a signed per-channel index (example: `pack_index.json`) containing `channel_id`, `latest` pointer(s), `pinned` pointer(s), optional `minimum_allowed_version`, optional `revoked` list. 
* **Cohort**: a subset of a channel (e.g., 1% of devices).

## Goals

* Minimize blast radius of a bad pack release (bad data, bad build, bad performance).
* Preserve offline correctness: never run a pack whose integrity fails verification.
* Support fast rollback without creating “downgrade vulnerabilities.”
* Maintain auditable linkage: build → release → distribution index → device state.

---

# 2. Release invariants (recommended)

These invariants should hold regardless of rollout strategy:

1. **Immutable pack identity**

   * A `pack_id` identifies immutable content.
   * Never “fix” a pack in place; publish a new pack.

2. **Fail-closed verification**

   * If signatures/hashes are declared, every distribution node and client verifies before activation.
   * If a signed channel index is used, it MUST be verified using pinned trust roots (fail-closed). 

3. **Activation gating + atomic activation**

   * Activate only after: integrity verified (if declared), manifest schema-valid, required files present, contract compatibility checks pass. 
   * Activation is an **atomic switch** from old to new (no partial activation). 

4. **Auditable release records**

   * Every activation is traceable to: `channel_id`, `pack_id`, `pack_version`, signer, timestamps (and Orgo release record if present).

5. **Explicit rollback/downgrade policy**

   * Downgrades are prevented by default (monotonic rule + revocation-aware rule). 
   * Any rollback is explicit and deterministic, not silent. 

---

# 3. Blue/Green releases (recommended default)

## 3.1 Concept

Maintain two production slots per channel:

* **Blue**: current active pack version
* **Green**: new candidate pack version

Devices or nodes only switch to Green once Green has been validated in production-like conditions.

## 3.2 Procedure

1. **Publish Green**

   * Orgo publishes a new release record referencing the new `pack_id` and channel scope.
   * Distribution replicates pack payloads to the Green slot (or to a staged cache location).

2. **Verify at rest**

   * Edge nodes/devices verify:

     * manifest schema validity,
     * payload hashes (if declared),
     * signatures (if present),
     * and (if used) `pack_index.json` signature for the channel (fail-closed). 
   * Any verification failure blocks activation.

3. **Warm-up / readiness checks**

   * Run pack-local checks:

     * query smoke tests (basic triple patterns),
     * join-cap breach behavior consistent with declared `join1_cap_policy` (`ERROR` vs `TRUNCATE`),
     * paging behavior consistent with declared `paging_mode` (`cursor` vs `offset`),
     * performance sanity (latency thresholds),
     * memory footprint thresholds. 
   * Log outcomes with correlation IDs.

4. **Switch traffic via signed channel index**

   * Update the channel’s `pack_index.json`:

     * set `latest` pointer(s) to the Green pack,
     * optionally add/update `pinned` to keep Blue as a known-good rollback target,
     * optionally update `minimum_allowed_version` and `revoked` if policy requires. 
   * Sign the updated index; distributors/clients verify it fail-closed. 
   * Devices activate Green using an atomic switch (no partial activation). 

5. **Post-switch monitoring**

   * Observe error codes, query latencies, resource usage.
   * If thresholds exceed, trigger explicit rollback (Section 5).

## 3.3 Advantages

* Simple mental model.
* Predictable rollback.
* Works well for offline package distribution where clients may activate asynchronously.

## 3.4 Pitfalls

* Storage overhead (two slots).
* Split-brain is expected (offline/asynchronous activation); auditability must be preserved via `channel_id` + `pack_version` + signer.

---

# 4. Canary releases (recommended for high-risk changes)

## 4.1 Concept

Roll out the new pack to a small cohort first, validate, then expand.

## 4.2 Procedure

1. **Create cohorts**

   * Example: 1% → 10% → 50% → 100%
   * Cohorts should be stable and deterministic (e.g., `hash(device_id) mod 100`).

2. **Canary stage via index**

   * Publish the pack (as in Blue/Green Step 1–2).
   * Update the channel pack index to include a staged/canary mechanism (implementation-defined), so that only the canary cohort resolves “current” to the new pack. (The distribution index is the primary control point for staged releases.) 
   * Sign and distribute the updated index; verify fail-closed. 

3. **Activation gating**

   * Devices verify integrity + schema validity + contract compatibility before activation. 

4. **Promote**

   * After the observation window, expand cohorts by updating the signed index (repeatable, auditable).

5. **Abort and rollback**

   * If health metrics fail, stop promotion and revert cohorts to the previous index pointer(s) (explicit rollback path; not silent). 

## 4.3 Metrics to watch

* Integrity verification failures (manifest/payload/index signature).
* Query error rates (especially join-cap breach behavior under `join1_cap_policy`).
* Median/P95 query latency for representative workloads.
* Memory pressure / OOM events on constrained devices.
* Pack download failures or excessive download size complaints.

## 4.4 Pitfalls

* Requires cohort assignment stability and a clean mechanism for cohort targeting in the index.
* Offline devices may “skip” cohorts; treat cohort targeting as best-effort and rely on activation-time verification + smoke tests.

---

# 5. Rollback and downgrade safety (critical)

Rollback is operationally necessary, but uncontrolled downgrade is a security risk.

## 5.1 Recommended rules

* **Default anti-downgrade rule**

  * Do not activate a pack whose `pack_version` is lower than the currently active one for the same channel unless an explicit rollback action/policy authorizes it. 
  * Clients should persist “highest activated” per channel and refuse lower versions by default. 

* **Rollback modes**

  * Prefer rollback to:

    * a **pinned** known-good pack, or
    * the **last-known-good** previously active pack still present and verified. 

* **Minimum version pin**

  * Maintain a per-channel `minimum_allowed_version` in the signed channel index.
  * Clients must not activate packs below this minimum (even if cached). 

* **Revocation**

  * If a pack is discovered invalid/vulnerable:

    * add it to `revoked` in the signed channel index (and/or other policy distribution mechanisms),
    * and, if needed, raise `minimum_allowed_version` to exclude it. 

(See also `07-security/downgrade-rollback-policy.md` and `06-integration/konnaxion-distribution-contract.md`.)  

---

# 6. Offline and constrained environments (practical notes)

## 6.1 Activation is local and asynchronous

Offline devices may:

* download a pack later than publication,
* activate it later than download.

Therefore:

* integrity verification MUST be done at activation time (fail-closed).
* store the activation decision with an audit marker (`channel_id` + `pack_id` + `pack_version` + timestamp).

If a device is offline and only has older packs available, and rollback is not explicitly authorized, the correct behavior is to **hold** rather than silently downgrade. 

## 6.2 Storage constraints

If devices cannot hold Blue+Green simultaneously:

* retain at minimum the previous pack’s manifest + signatures + any required index material needed to justify rollback decisions, or
* implement a “checkpoint pack” strategy: keep last known-good payloads for rollback where feasible.

## 6.3 Delta updates (optional)

If packs are large:

* use delta distribution at the transport layer,
* but ensure the final payload hashes match the declared pack manifest hashes.

---

# 7. Suggested control-plane data model (informative)

A practical Orgo/Konnaxion release system typically maintains:

* `BuildRecord(build_id, pack_id, source_kristal_id, compiler_version, config_hash, policies, inputs, signatures, created_at)`
* `ReleaseRecord(release_id, channel_id, pack_id, pack_version, signing_key_id, published_at, status)`  *(Orgo-level audit object)*
* `ChannelPackIndex(channel_id, latest, pinned, minimum_allowed_version?, revoked?, signature, signer_key_id, updated_at)`  *(serialized as `pack_index.json`)*
* `DeviceState(device_id, channel_id, active_pack_id, active_pack_version, highest_activated_pack_version, last_verified_at, verification_status)`

This is illustrative; exact implementation is system-specific.

---

# 8. Recommended “default strategy”

For most deployments:

* Use **Blue/Green** as the default.
* Add **Canary** for high-risk changes:

  * new portable policies (ordering, row-groups, filters),
  * large schema shifts,
  * new query extensions or contract versions,
  * new client versions.

---

# 9. Checklist

## Blue/Green

* [ ] Pack published + staged
* [ ] Verify manifest/payloads/signatures (fail-closed)
* [ ] Smoke tests pass (including declared `join1_cap_policy`, `paging_mode`)
* [ ] Update + sign `pack_index.json` (`latest` → Green; Blue pinned for rollback window)
* [ ] Monitor and rollback window defined
* [ ] `minimum_allowed_version` and `revoked` updated when needed

## Canary

* [ ] Cohorts stable and targetable
* [ ] Canary targeting represented via signed index updates
* [ ] Observation metrics defined
* [ ] Promotion steps defined (index updates are auditable)
* [ ] Abort/rollback is one operation (index revert + explicit rollback rules)
* [ ] Offline activation still verifies integrity + runs smoke tests
