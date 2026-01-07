# Downgrade and Rollback Policy (Kristal v3)

## Status
Draft

## Purpose
Define how Kristal v3 distributors and clients (notably Konnaxion offline package consumers and Orgo-controlled distribution flows) prevent:
- **downgrade attacks** (maliciously providing an older, vulnerable, or compromised artifact), and
- unsafe **rollbacks** (unintended reversion to older packs due to caching, sync conflicts, or operator error).

This policy is required to make signed Runtime Packs and Exchange releases safe to distribute at scale, especially in offline or intermittently connected environments.

## Scope
In scope:
- Versioning and monotonic update rules for Runtime Packs and Exchange releases
- Minimum metadata required to enforce rollback prevention
- Client enforcement behavior (accept/reject/hold)
- Offline-friendly mechanisms (no reliance on live services)
- Recovery paths (planned rollback, emergency rollback)

Out of scope:
- Key management and revocation mechanics (covered in key-management doc)
- Full content trust model beyond signature verification (covered elsewhere)
- UI/UX specifics (implementation-specific)

## Normative keywords
MUST, SHOULD, MAY as in RFC 2119.

## Core concepts

### Artifact types
- **Exchange release**: canonical, content-addressed, auditable reference.
- **Runtime Pack**: derived, offline-executable distribution payload.

Rollback/downgrade policies apply most critically to Runtime Packs, but SHOULD also apply to Exchange release pointers (e.g., “latest”).

### Threat model (minimum)
This policy defends against:
- A malicious distributor or compromised mirror serving old artifacts.
- A network attacker replaying older signed artifacts.
- Offline caches or sync conflicts reactivating older packs.
- Operator mistakes pushing the wrong version to a region.

This policy assumes:
- Signature verification exists and is fail-closed when present.
- Trust roots are pinned.
- Revocation lists may or may not be available offline (policy must handle both).

## Required metadata for enforcement
Artifacts intended for distribution MUST include, in their manifest (or a distribution index manifest):

1. **artifact_id**
   - Content-addressed id (e.g., `kristal_id` or pack id).
2. **artifact_type**
   - `exchange` or `runtime_pack`.
3. **release_id** (monotonic)
   - A monotonically increasing identifier (see below).
4. **build_id**
   - Unique id for build lineage and audit.
5. **created_at**
   - Timestamp for debugging/audit (not used alone for monotonicity).
6. **channel**
   - Distribution channel identifier (e.g., `prod`, `region-ca-qc`, `tenant-x-prod`).
7. **signer_key_id**
   - The signing key identifier.
8. **parent_release_ids** (recommended)
   - For lineage/merge cases (optional but useful).
9. **revocation_epoch** (recommended)
   - A small integer that increments whenever revocation policy changes materially (e.g., key compromise event).

### Monotonic release_id formats
An implementation MUST choose one monotonic scheme and enforce it consistently per channel.

Allowed schemes:
- **Integer sequence**: `release_id = 10293` (recommended).
- **Semantic version**: `v3.1.4` with strict ordering rules (harder operationally).
- **Hybrid**: integer sequence + human label.

Requirement:
- Comparison MUST be deterministic and documented.

## Distribution index (“what to install”)
Clients SHOULD not rely on “latest” pointers embedded in artifacts alone. Instead, they SHOULD consume a signed **distribution index** per channel.

A distribution index MUST include:
- channel id
- current release_id
- current artifact_id
- optional staged releases (canary)
- signature and signer identity

This index is the primary control point for controlled rollout and rollback.

## Client enforcement rules (mandatory)

### 1) Basic rule: never activate a lower release_id
A client MUST NOT activate an artifact with a lower `release_id` than the highest previously activated release in the same channel, unless a **policy-authorized rollback** is present (see below).

Clients MUST persist:
- `highest_activated_release_id[channel]`
- `highest_seen_release_id[channel]` (recommended)
- last successful verification metadata

### 2) Signature verification first
Before comparing release_ids, clients MUST:
- verify the signature chain to a pinned trust root (when signatures are present)
- verify integrity fields fail-closed
- consult revocations if policy requires and data is available

If verification fails:
- the artifact MUST be rejected regardless of release_id.

### 3) Channel isolation
Rollback and downgrade decisions MUST be scoped by channel.
Example:
- `tenantA-prod` is independent from `tenantA-staging`.

### 4) Handling “equal release_id”
If a client receives an artifact with the same `release_id` but different `artifact_id`:
- The client MUST treat this as a hard error unless an explicit **reissue flag** is present in the signed distribution index.
- This prevents silent substitution attacks.

## Policy-authorized rollback (controlled rollback)
Rollbacks are sometimes necessary (bad pack release). They must be explicit and signed.

### Rollback authorization artifact
A rollback MUST be authorized by a signed “rollback authorization” record, either embedded in the distribution index or distributed alongside it, containing:

- channel id
- `authorized_rollback_to_release_id`
- `reason` (string)
- `issued_at`
- `expires_at` (recommended)
- signature by an authority key (root or intermediate)

Clients MUST:
- verify authorization signature
- ensure the authorization is within validity window
- ensure rollback target is not revoked/blocked by policy
- record that rollback occurred (audit)

### Rollback activation rules
When rollback authorization is present:
- clients MAY activate the rollback target release_id even if lower than previously activated
- but MUST update local state to:
  - keep `highest_seen_release_id` unchanged (so the client remembers newer releases exist)
  - record `current_active_release_id` separately from `highest_activated_release_id` (recommended)

This enables recovery and prevents oscillation.

## Emergency rollback for compromise response
If a key compromise occurs, a rollback may be used to move to a “safe older” release, but only if:
- the older release was signed by non-compromised keys (or is otherwise still trusted), and
- revocation policy does not invalidate it.

In compromise scenarios, clients SHOULD prefer:
- a reissued pack at the same release_id with a reissue flag (if allowed), or
- a new higher release_id that contains the rollback fix.

## Offline and sync edge cases

### Offline clients without index updates
If a client is offline and only has cached artifacts:
- it MUST NOT activate a lower release_id than previously active in that channel unless rollback authorization is cached and valid.
- if only an older artifact is available and no valid rollback authorization exists, the client SHOULD enter a “hold” state rather than silently downgrade.

### Sync conflicts (multi-device / multi-node)
When Orgo/Konnaxion sync introduces conflicting “current” pointers:
- The client MUST choose the highest valid release_id according to channel rules, unless rollback authorization exists.
- Conflicts SHOULD be logged with correlation ids.

## Operational rollout patterns (recommended)
- **Canary releases**: distribution index lists canary cohorts and their current release_id.
- **Blue/green**: clients have active+standby pack slots; activation only occurs after verification.
- **Staged rollouts**: progressively advance `current release_id` in the signed index.

## Logging and observability (recommended)
Clients and distributors SHOULD emit structured events:
- `PACK_VERIFIED` (artifact_id, release_id, signer_key_id)
- `PACK_REJECTED` (reason: verification_failed | downgrade_blocked | revoked_key | index_signature_failed)
- `DOWNGRADE_BLOCKED`
- `ROLLBACK_AUTH_ACCEPTED`
- `ROLLBACK_AUTH_REJECTED`
- `PACK_ACTIVATED` (active_release_id, highest_seen_release_id)

Include: build_id, channel, tenant, device/node id.

## Minimum acceptance criteria
A deployment satisfies this policy if:
1. Clients persist per-channel highest release_id and block downgrades.
2. Rollbacks require an explicit signed authorization record.
3. “Same release_id but different artifact_id” is rejected unless explicitly reissued via signed index.
4. Signature verification is enforced before activation.
5. Offline clients fail safe (hold rather than downgrade silently).

## Open questions (to finalize)
- Whether `release_id` is global per tenant or per channel/region.
- Whether “reissue” is allowed, and which authority can authorize it.
- How long rollback authorizations remain valid in constrained offline environments.
- Whether to require a signed distribution index in all environments (recommended for production).
