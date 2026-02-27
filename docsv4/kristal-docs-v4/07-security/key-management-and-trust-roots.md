# Key Management and Trust Roots (Kristal v3)

## Status
Draft (implementation-ready defaults; future profiles may extend)

## Purpose
Define how Kristal v3 artifacts (Exchange and Runtime Packs) are signed, how trust roots are established, and how verifiers validate signatures in a **fail-closed** manner when integrity declarations are present.

This document is security and operations focused. It does **not** change Kristal’s core content-addressed identity rules; it defines the trust model around distribution and verification.

## Scope
In scope:
- Key types and roles (root, intermediate, signing)
- Supported signature algorithms (baseline + optional compatibility)
- Trust root pinning models (tenant-scoped and environment-scoped)
- Signature verification requirements for distributors and clients
- Rotation, revocation, and compromise handling
- Minimum manifest fields for signer identity and key metadata
- Runtime Pack signing scope (manifest + referenced file integrity)

Out of scope:
- Network PKI / CA integration details (may be deployment-specific)
- Secrets storage implementation (HSM/KMS/etc.)—recommended but not mandated

## Normative keywords
MUST, SHOULD, MAY as in RFC 2119.

## Design goals
- **Fail-closed integrity**: if an artifact declares signatures/hashes, verification failures are hard errors.
- **Tenant isolation**: tenants may use independent trust roots and keys.
- **Offline viability**: verification must be possible without network access.
- **Operational safety**: rotation, rollback prevention, and compromise response are first-class.

## Supported signature algorithms
Baseline:
- Verifiers MUST support **Ed25519** (`"ed25519"`).

Compatibility (optional):
- Verifiers MAY support **RSA-PSS with SHA-256** (`"rsa-pss-sha256"`) for deployments with legacy constraints.

Deployments MUST document which compatibility algorithms are enabled. Artifacts SHOULD prefer Ed25519 when available.

## Key roles

### 1) Trust Root (Root Key)
A long-lived key that anchors trust for a tenant or environment.

Requirements:
- Trust roots MUST be pin-able (fingerprint) by distributors and clients.
- Trust roots SHOULD be offline or highly protected (HSM/KMS).
- Trust roots SHOULD NOT sign artifacts directly; they should sign intermediates and/or authority registry updates.

### 2) Intermediate / Release Authority Key
A medium-lived key used to sign artifact signing keys or release signing certificates/attestations.

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
1. Validate the signature(s) against the artifact’s canonical hashing rules and declared signing target.
2. Validate signer identity and key chain to a pinned trust root (directly or via intermediates).
3. Enforce validity windows.
4. Enforce revocation rules as available offline (see Revocation).

If any step fails, verification MUST fail closed.

### What is signed (signing target)
The signed payload MUST be the canonical hash of the artifact’s **hash target** (i.e., the portion that defines identity/integrity), excluding signatures themselves.

- **Exchange**
  - The signing target MUST be the canonical hash of the Exchange identity payload, excluding any `signatures` fields.
  - The manifest MUST make it unambiguous how the payload hash was computed.

- **Runtime Pack**
  - The signing target MUST cover:
    1) the canonical hash of the **pack manifest**, excluding any `signatures` fields, AND
    2) the integrity of all referenced payload files as recorded in the manifest (e.g., file hashes).
  - Concretely: a Runtime Pack manifest MUST include hashes for every payload file it references, and the pack signature(s) MUST be computed over the manifest that includes those hashes.
  - “Manifest-only” signing MAY exist only as a deployment-specific profile; it MUST NOT be the default.

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
- `signature`: base64url (or multibase) signature bytes
- `chain`: optional embedded cert/attestation chain (root excluded if pinned externally)

Exact envelope shape is implementation-defined, but all required semantic fields MUST be representable. Fields used for signature verification MUST NOT be ambiguous.

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

## Authority Registry (recommended policy data artifact)
Deployments SHOULD maintain a pinned, versioned **Authority Registry** artifact that:
- enumerates accepted trust roots (active/deprecated/blocked),
- defines authority classes (e.g., personal vs recognized) and applicable scopes,
- references revocation lists and required validation profiles when relevant.

When present, distributors/clients SHOULD treat the Authority Registry as the authoritative policy input for verification decisions.

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
- Mark affected key ids as revoked (see Revocation).
- Stop issuing new artifacts signed by the compromised key immediately.
- Re-issue current packs signed with new keys.
- Apply downgrade/rollback protections to prevent clients from accepting older compromised artifacts.

## Revocation (offline-friendly)
Because clients may be offline, revocation must work without live OCSP/CRLs.

Recommended approach:
- Maintain a signed **revocation list** artifact per tenant/environment:
  - `revocations.json` (content-addressed, signed by trust root or intermediate)
  - Contains revoked `key_id`s, reasons, and effective timestamps.
- Distribute revocation lists alongside Runtime Packs and/or through Orgo/Konnaxion sync channels.

Clients MUST:
- Consult the latest available revocation list before accepting a signature, when a revocation list is present.
- Fail closed if policy requires revocation checking and no revocation list is available.

Deployments MUST define the revocation policy level per environment (e.g., required in prod; optional in dev).

## Verification responsibilities by component

### Orgo (control plane)
- MUST record: build_id, artifact ids, signer key_id, signature metadata.
- SHOULD enforce “no publish without signature” in environments where signatures are required.
- SHOULD manage rotation schedules and revocation list publication.
- SHOULD publish/update the Authority Registry (if used) as pinned policy data.

### Konnaxion (distribution + client UX)
- MUST verify signatures when present before activating a pack.
- MUST pin trust roots per tenant/environment (or via Authority Registry policy).
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
2. Trust roots are pinned and enforceable offline (pin sets supported).
3. Key rotation is supported (overlap windows + key ids recorded).
4. Compromise response exists (revocation list distribution + re-issue).
5. Downgrade protection is implemented for pack activation (see rollback policy doc).

## Future optional profiles (non-default)
- Transparency logs (append-only) for additional auditability and compromise detection.