# Downgrade and Rollback Policy (Kristal v3)

## Status

Draft

## Purpose

Define how Kristal v3 distributors and clients (notably Konnaxion offline package consumers and Orgo-controlled distribution flows) prevent:

* **downgrade attacks** (maliciously providing an older, vulnerable, or compromised artifact), and
* unsafe **rollbacks** (unintended reversion to older packs due to caching, sync conflicts, or operator error).

This policy is required to make signed Runtime Packs and Exchange releases safe to distribute at scale, especially in offline or intermittently connected environments. 

## Scope

In scope:

* Versioning and monotonic update rules for Runtime Packs and Exchange release pointers
* Minimum metadata required to enforce downgrade/rollback prevention
* Client enforcement behavior (accept/reject/hold)
* Offline-friendly mechanisms (no reliance on live services at activation time)
* Recovery paths (planned rollback, emergency rollback)

Out of scope:

* Key management and revocation mechanics (covered in key-management doc)
* Full content trust model beyond signature verification (covered elsewhere)
* UI/UX specifics (implementation-specific)

## Normative keywords

MUST, MUST NOT, SHOULD, SHOULD NOT, MAY as in RFC 2119.

---

## Core concepts

### Artifact types (normative)

* **Exchange (payload)**: canonical, content-addressed knowledge artifact (`kristal_id`) referenced by releases/pointers.
* **Runtime Pack**: derived, offline-executable distribution payload (bundle) identified by `pack_id` (preferred) plus `pack_version` (monotonic per channel). 

Rollback/downgrade policies apply most critically to Runtime Packs, but SHOULD also apply to Exchange release pointers (e.g., “latest”). 

### Terminology mapping (normative)

To avoid cross-doc ambiguity:

* **`release_id` (this policy)** = the monotonic **channel-scoped** version identifier.
* For Runtime Packs: **`release_id` == `pack_version`**. 
* For Exchange pointers: **`release_id`** refers to the monotonic identifier used by the distribution index for that channel (implementation-defined; MUST be deterministic).

### Threat model (minimum)

This policy defends against:

* A malicious distributor or compromised mirror serving old artifacts.
* A network attacker replaying older signed artifacts.
* Offline caches or sync conflicts reactivating older packs.
* Operator mistakes pushing the wrong version to a region.

This policy assumes:

* Signature verification exists and is fail-closed when present. 
* Trust roots are pinned per channel and can be verified offline. 
* Revocation data may or may not be available offline; the policy must handle both.

---

## Required metadata for enforcement

### A) Runtime Pack bundle identifiers (mandatory)

Each Runtime Pack bundle MUST be uniquely identified by:

* `pack_id` (content-addressed, preferred) OR deterministic `pack_content_id`
* `source_kristal_id` (the Exchange `kristal_id` used to compile the pack)
* `build_id` (compiler build id)
* `pack_version` (monotonic within a channel) 

### B) Channel and trust scoping (mandatory)

Packs MUST be distributed via **channels** that are scoped at minimum by:

* `tenant_id`
* `environment` (e.g., `prod`, `staging`)
* optional `region` or `audience_segment` 

A client MUST NOT mix trust roots or activation rules across channels unless explicitly configured. 

### C) Channel distribution index (recommended; mandatory if used)

Clients SHOULD consume a signed channel index (example: `pack_index.json`) as the primary “what to install” control plane. 

If present, the index MUST be verified using the channel’s trust roots (fail-closed). 

A channel index SHOULD contain, at minimum:

* `channel_id`
* `latest` pointer(s)
* `pinned` pointer(s)
* optional `minimum_allowed_version`
* optional `revoked` list
* optional staged pointers/cohorts (canary)
* signature and signer identity 

### D) Timestamps and signatures (normative constraints)

* Timestamps (e.g., `created_at`) MAY be present for audit/debugging but MUST NOT affect any content-addressed IDs. 
* Signature/attestation material MUST be excluded from hash targets wherever it appears. 

---

## Monotonic version formats (for `pack_version` / `release_id`)

An implementation MUST choose one monotonic scheme and enforce it consistently per channel.

Allowed schemes:

* **Integer sequence**: recommended.
* **Semantic version**: allowed with strict deterministic ordering rules.
* **Hybrid**: integer sequence + human label.

Requirement:

* Comparison MUST be deterministic and documented.

---

## Client enforcement rules (mandatory)

### 1) Basic rule: never activate a lower `pack_version`

A client MUST NOT activate a Runtime Pack with a lower `pack_version` than the highest previously activated version in the same channel, unless an explicit rollback authorization permits it. 

Clients MUST persist per channel:

* `highest_activated_pack_version[channel]`
* `highest_seen_pack_version[channel]` (recommended)
* `current_active_pack_version[channel]` (recommended)
* last successful verification metadata (e.g., signer id, index verification state)

### 2) Verification before any activation decision (fail-closed)

Before comparing versions or switching active packs, clients MUST:

* verify the channel index signature (if used)
* verify pack manifest signature(s) and declared integrity material (if present)
* fail closed on any verification failure  

Trust roots MUST be pinned per channel and MUST NOT require network fetching at activation time. 

### 3) Atomic activation

Activation MUST be an atomic switch from old pack to new pack (no partial activation). 

### 4) Channel isolation

Downgrade and rollback decisions MUST be scoped by channel (tenant/env/region/audience). 

### 5) Handling “equal version but different identity”

If a client receives a pack with the same `pack_version` but different `pack_id`:

* The client MUST treat this as a hard error unless an explicit **reissue flag** is present in the signed channel index.
* This prevents silent substitution attacks.

### 6) Revocation-aware blocking (if revocations are provided)

If a verified channel index includes a `revoked` list:

* clients MUST NOT activate any pack listed as revoked (even if cached). 

If a `minimum_allowed_version` is present in a verified index:

* clients MUST NOT activate packs below this minimum, even if cached. 

---

## Policy-authorized rollback (controlled rollback)

Rollback is operationally necessary but MUST be explicit and signed.

### Rollback authorization record

A rollback MUST be authorized by a signed rollback authorization record, either embedded in the channel index or distributed alongside it, containing:

* `channel_id`
* `authorized_rollback_to_pack_version` (i.e., a target `pack_version`)
* optional `target_pack_id` (recommended, to prevent ambiguity)
* `reason`
* `issued_at`
* `expires_at` (recommended)
* signature by an authority key (root or intermediate)

Clients MUST:

* verify authorization signature
* ensure it is within the validity window
* ensure rollback target is not revoked/blocked by policy
* record that rollback occurred (audit)

### Rollback activation rules (state update)

When rollback authorization is present:

* clients MAY activate the rollback target even if it is lower than the highest previously activated version
* but MUST:

  * keep `highest_seen_pack_version` unchanged (so the client remembers newer releases exist)
  * record `current_active_pack_version` separately from `highest_activated_pack_version` (recommended)

---

## Emergency rollback for compromise response

If a key compromise occurs, a rollback may be used to move to a “safe older” pack only if:

* the older pack was signed by non-compromised keys (or is otherwise still trusted), and
* revocation policy does not invalidate it.

Clients SHOULD prefer:

* a reissued pack at the same `pack_version` with a reissue flag (if allowed), or
* a new higher `pack_version` that contains the rollback fix.

---

## Offline and sync edge cases

### Offline clients without index updates

If a client is offline and only has cached artifacts:

* it MUST NOT activate a lower `pack_version` than previously active in that channel unless a cached rollback authorization is present and valid.
* if only an older pack is available and no valid rollback authorization exists, the client SHOULD enter a **hold** state rather than silently downgrading. 

### Sync conflicts (multi-device / multi-node)

When Orgo/Konnaxion sync introduces conflicting “current” pointers:

* the client MUST choose the highest valid `pack_version` according to channel rules unless rollback authorization exists.
* conflicts SHOULD be logged with correlation ids.

---

## Operational rollout patterns (recommended)

* **Canary releases**: index lists cohorts/staged pointers; clients still gate activation locally with verification + smoke tests. 
* **Blue/green**: clients keep active + standby slots; activation is atomic after verification. 
* **Staged rollouts**: progressively advance `latest` (or cohort pointers) in the signed channel index.

(See also `08-ops/release-strategies.md` and `06-integration/konnaxion-distribution-contract.md`.)  

---

## Logging and observability (recommended)

Clients and distributors SHOULD emit structured events:

* `PACK_VERIFIED` (pack_id, pack_version, signer_key_id)
* `PACK_REJECTED` (reason: verification_failed | downgrade_blocked | revoked | index_signature_failed)
* `DOWNGRADE_BLOCKED`
* `ROLLBACK_AUTH_ACCEPTED`
* `ROLLBACK_AUTH_REJECTED`
* `PACK_ACTIVATED` (current_active_pack_version, highest_seen_pack_version)

Include: build_id, channel_id, tenant_id, device/node id.

---

## Minimum acceptance criteria

A deployment satisfies this policy if:

1. Clients persist per-channel highest version and block downgrades.
2. Rollbacks require an explicit signed authorization record.
3. “Same version but different identity” is rejected unless explicitly reissued via signed index.
4. Verification is enforced before activation (fail-closed) and trust roots are pinned offline.
5. Offline clients fail safe (hold rather than downgrade silently). 

---

## Open questions (to finalize)

* Whether `pack_version` is global per tenant or strictly per channel/region/audience segment.
* Whether “reissue” is allowed, and which authority can authorize it.
* How long rollback authorizations remain valid in constrained offline environments.
* Whether a signed channel index (`pack_index.json`) is required in all environments (recommended for production).
