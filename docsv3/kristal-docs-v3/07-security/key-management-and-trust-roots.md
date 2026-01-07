# Key Management and Trust Roots (Kristal v3)

## Status
Draft

## Purpose
Define how Kristal v3 artifacts (Exchange and Runtime Packs) are signed, how trust roots are established, and how verifiers validate signatures in a **fail-closed** manner when integrity declarations are present.

This document is security and operations focused. It does **not** change Kristal’s core content-addressed identity rules; it defines the trust model around distribution and verification.

## Scope
In scope:
- Key types and roles (root, intermediate, signing)
- Trust root pinning models (tenant-scoped and environment-scoped)
- Signature verification requirements for distributors and clients
- Rotation, revocation, and compromise handling
- Minimum manifest fields for signer identity and key metadata

Out of scope:
- Specific cryptographic algorithm choices beyond “supported set” guidance
- Network PKI / CA integration details (may be deployment-specific)
- Secrets storage implementation (HSM/KMS/etc.)—recommended but not mandated

## Normative keywords
MUST, SHOULD, MAY as in RFC 2119.

## Design goals
- **Fail-closed integrity**: if an artifact declares signatures/hashes, verification failures are hard errors.
- **Tenant isolation**: tenants may use independent trust roots and keys.
- **Offline viability**: verification must be possible without network access.
- **Operational safety**: rotation, rollback prevention, and compromise response are first-class.

## Key roles

### 1) Trust Root (Root Key)
A long-lived key that anchors trust for a tenant or environment.

Requirements:
- Trust roots MUST be pin-able (fingerprint) by distributors and clients.
- Trust roots SHOULD be offline or highly protected (HSM/KMS).
- Trust roots SHOULD NOT sign artifacts directly; they should sign intermediates.

### 2) Intermediate / Release Authority Key
A medium-lived key used to sign artifact signing keys or release signing certificates.

Requirements:
- Intermediates MUST be signed by a trust root.
- Intermediates MUST include validity windows.
- Intermediates SHOULD be rotated on a scheduled basis (e.g., quarterly).

### 3) Artifact Signing Key (Pack/Exchange Signing Key)
A shorter-lived key used to sign Exchange commits and Runtime Packs.

Requirements:
- Signing keys MUST be scoped (tenant + environment + artifact type).
- Signing keys SHOULD be rotated regularly (e.g., monthly) and on compromise events.
- Signing keys MUST be auditable (key id, creation, rotation history).

## Trust models

### Model A: Tenant-pinned trust roots (recommended)
Each tenant has its own trust root (or root set). Clients pin the tenant’s root fingerprint(s).

Advantages:
- Strong tenant isolation
- Works offline
- Limits blast radius

### Model B: Environment-pinned trust roots
A deployment pins roots by environment (e.g., dev/stage/prod). Tenants are separated by access control rather than separate roots.

Advantages:
- Simpler operations for small deployments
- Still supports offline verification

Risk:
- Larger blast radius if the root is compromised.

Implementations MUST document which model is used and how pins are distributed.

## Artifact signature requirements

### When signatures are present
If an artifact declares signatures, verifiers MUST:
1. Validate the signature(s) against the artifact’s canonical hashing rules.
2. Validate signer identity and key chain to a pinned trust root.
3. Enforce validity windows.
4. Enforce revocation rules as available offline (see revocation section).

If any step fails, verification MUST fail closed.

### What is signed
The signed payload MUST be the canonical hash of the artifact content:
- Exchange: hash of canonical JSON (JCS), excluding signatures.
- Runtime Pack: hash of the pack manifest’s canonical JSON (JCS) AND (optionally) hashes of referenced files recorded in the manifest.

The signing target MUST be unambiguous and recorded in the manifest.

### Recommended signature envelope fields
Artifacts that are signed SHOULD include a `signatures[]` section in a manifest-like structure with:

- `sig_id`: unique signature id
- `alg`: signature algorithm identifier (e.g., `"ed25519"`, `"rsa-pss-sha256"`)
- `key_id`: signer key identifier
- `signer`: human/organization identifier (string or DID-like)
- `created_at`: timestamp
- `expires_at`: timestamp (optional)
- `payload_hash`: `{ alg: "sha256", value: "<hex>" }`
- `signature`: base64 (or multibase) signature bytes
- `chain`: optional embedded cert/attestation chain (root excluded if pinned externally)

Exact envelope shape is implementation-defined, but all required semantic fields MUST be representable.

## Trust root pinning and distribution

### Pins
A pin is a stable fingerprint of a trust root public key.

Requirements:
- Pins MUST be distributed to clients out-of-band or via a secure bootstrap channel.
- Pins MUST be stored and enforced locally for offline verification.
- Pins SHOULD support multiple active roots to enable rotation.

### Pin sets
Clients and distributors SHOULD support a pin set:
- `active_roots[]`
- `deprecated_roots[]` (accepted for a grace period if allowed by policy)
- `blocked_roots[]` (explicitly rejected)

Policies around acceptance of deprecated roots MUST be documented.

## Key rotation

### Planned rotation
- Trust roots rotate rarely (years) and require careful migration.
- Intermediates rotate periodically (months/quarters).
- Signing keys rotate frequently (weeks/months).

Rotation requirements:
- Rotation MUST support overlap windows (old + new valid simultaneously).
- Clients MUST support multiple pinned roots/intermediates during overlap.
- Artifacts SHOULD record which key id signed them for auditability.

### Emergency rotation (compromise)
On compromise:
- Mark affected key ids as revoked (see revocation).
- Stop issuing new artifacts signed by the compromised key immediately.
- Re-issue current packs signed with new keys.
- Apply downgrade/rollback protections to prevent clients from accepting older compromised artifacts.

## Revocation (offline-friendly)
Because clients may be offline, revocation must work without live OCSP/CRLs.

Recommended approach:
- Include a signed **revocation list** artifact per tenant/environment:
  - `revocations.json` (content-addressed, signed by trust root or intermediate)
  - Contains revoked `key_id`s, reasons, and effective timestamps.
- Distribute revocation lists alongside Runtime Packs and/or through Orgo/Konnaxion sync channels.

Clients MUST:
- Consult the latest available revocation list before accepting a signature, when a revocation list is present.
- Fail closed if policy requires revocation checking and no revocation list is available.

## Verification responsibilities by component

### Orgo (control plane)
- MUST record: build_id, artifact ids, signer key_id, signature metadata.
- SHOULD enforce “no publish without signature” in environments where signatures are required.
- SHOULD manage rotation schedules and revocation list publication.

### Konnaxion (distribution + client UX)
- MUST verify signatures when present before activating a pack.
- MUST pin trust roots per tenant/environment.
- MUST enforce rollback/downgrade policy (see separate doc).

### Architect (renderer)
- SHOULD verify inputs are validated and (where required) signed before rendering.
- MUST include traceability, but does not need to re-sign artifacts unless it produces distributable outputs.

### SenTient (resolver)
- Not typically a signer.
- SHOULD sign resolver outputs only if the deployment treats resolver output as a distributable artifact (rare).

## Minimum security acceptance criteria
A deployment meets minimum Kristal v3 security criteria if:
1. Signed artifacts are verified fail-closed when signatures are present.
2. Trust roots are pinned and enforceable offline.
3. Key rotation is supported (overlap windows + key ids recorded).
4. Compromise response exists (revocation list distribution + re-issue).
5. Downgrade protection is implemented for pack activation (see rollback policy doc).

## Open questions (to finalize)
- Supported signature algorithms set (Ed25519 recommended baseline vs RSA compatibility needs)
- Whether Runtime Packs sign:
  - only the manifest, or
  - manifest + referenced file hashes (stronger)
- Required revocation policy level for “offline-only” constrained environments
- Whether to support transparency logs (append-only) as a future optional profile
