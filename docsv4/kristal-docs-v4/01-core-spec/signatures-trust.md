# 01-core-spec/signatures-trust.md

## Status

Draft (v3)

## Purpose

This document specifies the **normative** rules for:

* signature and attestation structures used in Kristal v4 artifacts
* what is signed (and how signing inputs are constructed)
* verification requirements, including **fail-closed** semantics
* trust-root and key identity (`key_id` / KID), and offline verification expectations
* minimum interoperability requirements (mandatory-to-implement algorithms)

This document applies to:

* **Kristal Exchange** artifacts
* **Runtime Pack** manifests (and, optionally, other derived artifacts)

---

## 1. Terminology

* **Artifact**: a JSON object (Exchange or Runtime Pack manifest).
* **Content identity**: the artifact’s content-addressed identifier, computed per `01-core-spec/ids-canonicalization-hashing.md`:
  * Exchange: `kristal_id`
  * Runtime Pack manifest: `runtime_pack_id`
* **Hash target**: the JSON object that is canonicalized and hashed to produce content identity (with signature overlays excluded), per `01-core-spec/ids-canonicalization-hashing.md`.
* **Signature**: a cryptographic proof that binds an issuer key to an artifact’s content identity.
* **Attestation**: a signature produced by a third party (auditor, publisher, distributor) over the same content identity.
* **Trust root**: a pinned set of public keys (or certificates) trusted by a verifier for a given tenant/environment.
* **Fail-closed**: if an artifact declares required signatures, verification failure is a hard error; the artifact MUST NOT be treated as valid.

Normative keywords: MUST, MUST NOT, SHOULD, SHOULD NOT, MAY.

---

## 2. Core principles

1. **Content IDs are not signatures.**
   Content identity proves content equivalence; signatures prove issuer authenticity/authorization.

2. **Signatures MUST NOT be included in the content-hash target.**
   Signatures are overlays; the content identity is computed over the artifact with signature fields excluded (see IDs/canonicalization spec).

3. **Verification must be offline-capable.**
   A verifier must be able to verify a signed artifact with no network access, given the artifact and the relevant trust roots.

4. **If declared, integrity MUST be enforced.**
   Presence of required signatures implies fail-closed behavior.

---

## 3. Signature envelope

### 3.1 Where signatures live (normative)

Artifacts that support signatures MUST include a top-level field:

* `signatures`: an array of signature objects

If `signatures` is absent or empty, the artifact is considered **unsigned**.

**Placement constraint:** signature objects MUST be carried only in the top-level `signatures` field. If signature-like fields appear elsewhere, verifiers MUST exclude them from the hash target and SHOULD reject the artifact as non-conformant.

### 3.2 Signature object schema (normative)

Each entry in `signatures[]` MUST be an object with:

* `key_id` (string): key identifier (KID) used to locate the issuer public key in the trust store
* `alg` (string): signing algorithm identifier
* `signature` (string): signature bytes encoded as base64url (no padding)
* `created_at` (string, optional): ISO 8601 / RFC 3339 timestamp
* `required` (boolean, optional): defaults to `true`

Optional fields (profiles / non-normative unless explicitly enabled):

* `purpose` (string): e.g., `"publisher"`, `"auditor"`, `"distributor"`
* `signer` (object): human-readable signer metadata (e.g., `{ "name": "...", "org": "..." }`)
* `expires_at` (string): ISO 8601 / RFC 3339 timestamp
* `notes` (string)
* `x5c` (array of strings): X.509 certificate chain (base64 DER) (optional profile)

### 3.3 Mandatory-to-implement algorithms (interop)

To ensure interoperability, v3 implementations MUST support verification for:

* `alg = "ed25519"`
* content identity hash algorithm `sha256` (as defined by v3 core ID rules)

Implementations MAY support additional algorithms, but MUST NOT claim v3 conformance if they cannot verify Ed25519 signatures over v3 SHA-256 content identities.

---

## 4. Signing input construction

### 4.1 Goals

The signing input MUST:

* bind the issuer to the artifact’s **content identity**
* prevent ambiguous “what exactly was signed” situations
* be reconstructable deterministically by verifiers from the artifact alone

### 4.2 Signing input (normative)

For each signature object `S` in `A.signatures[]`, the signing input MUST be the **raw digest bytes** of the artifact’s content identity:

* **Exchange**: decode the digest portion of `A.kristal_id` (SHA-256) into 32 raw bytes
* **Runtime Pack manifest**: decode `A.runtime_pack_id` (SHA-256 hex) into 32 raw bytes

**Identity recomputation requirement (mandatory):**
Before verifying any signatures, verifiers MUST recompute the artifact’s content identity from the artifact itself (per `01-core-spec/ids-canonicalization-hashing.md`, including hash exclusions). If the recomputed identity does not match the declared identity field, verification MUST fail (hard error if any required signature is present).

### 4.3 Signature computation (normative)

For `alg="ed25519"`:

* signature input MUST be the raw 32 bytes described in Section 4.2
* `signature` MUST be base64url-encoded signature bytes (no padding)

---

## 5. Verification rules (fail-closed)

### 5.1 Verification procedure (normative)

Given an artifact `A`:

1. If `A.signatures` is absent or empty: artifact is **unsigned**.
2. Otherwise:
   1) Recompute `A`’s content identity from `A` (excluding signatures) and compare to the declared identity.  
      *If mismatch: verification fails.*
   2) For each signature object `S` in `A.signatures`:
      * Validate required fields exist and are well-formed.
      * Resolve the public key for `S.key_id` from the verifier’s trust store.
      * Verify `S.signature` under `S.alg` over the signing input defined in Section 4.2.
3. Apply required/optional semantics:
   * If `S.required` is true or missing, it is required.
   * If any required signature fails, verification MUST fail.

### 5.2 Fail-closed semantics (normative)

If an artifact contains any signature object that is **declared** (present in `signatures[]`) and that signature is **required**, then:

* any verification failure (malformed signature, missing key, identity mismatch, bad signature, unsupported alg) MUST be treated as a hard error
* the artifact MUST NOT be accepted for compilation, execution, distribution, or rendering

Unsigned artifacts MAY be accepted depending on deployment policy, but must be clearly treated as unsigned.

### 5.3 Partial verification (normative)

Verifiers MUST NOT silently skip signatures they cannot process if those signatures are required.

If a verifier does not support the algorithm referenced by a required signature:

* verification MUST fail (hard error)

---

## 6. Trust roots, key identity, and multi-tenancy

### 6.1 Trust roots (normative)

Each verifier MUST maintain a trust store containing:

* a set of trusted public keys (or certificates) keyed by `key_id`

Trust stores MAY be:

* tenant-scoped
* environment-scoped (prod/staging)
* device-scoped (pinned to an app install)

### 6.2 `key_id` format (recommended)

Recommended `key_id` formats:

* stable string key IDs (e.g., `"orgo-prod-ed25519-2026-01"`)
* or key fingerprints (e.g., multibase/multihash), provided the encoding is stable

### 6.3 Key rotation (recommended)

* allow overlapping validity windows where old and new keys are trusted
* do not remove a key until artifacts signed by it are no longer accepted in that environment
* prefer short-lived signing keys with longer-lived root keys (optional profile)

---

## 7. What signatures authorize (deployment policy)

This spec binds a signature to a content identity. Whether that signature is “enough” to:

* publish Exchange into a registry
* distribute a Runtime Pack
* display “verified” badges

is governed by product policy.

Recommended minimum policy:

* **Orgo**: enforce “no compile on validation fail” and signature policies for publish/release.
* **Konnaxion**: verify signatures before activating packs when signatures are present; pin trust roots per tenant/environment.
* **Architect**: render only from validated artifacts; surface signature/issuer metadata where relevant.

---

## 8. Optional attestation layers (profiles)

Implementations MAY support multiple signatures in `signatures[]`:

* publisher signature (who released it)
* auditor signature (third-party review)
* distributor signature (who served it to devices)

A deployment MAY require:

* at least one valid required signature
* or a specific `purpose` signature (e.g., `purpose="publisher"`)

If such requirements exist, they MUST be expressed in a deployment policy document (not in the artifact core schema), to keep the standard’s normative surface area small.

---

## 9. Conformance tests

A v3 implementation MUST provide test vectors that cover:

* positive verification cases (valid identity + valid signature)
* negative cases (artifact tampering causing identity mismatch; wrong key; malformed signature; unsupported alg on required signature)
* fail-closed behavior (required signature failure blocks acceptance)

Test vectors SHOULD include:

* Exchange artifacts and Runtime Pack manifests
* multiple `key_id` resolutions
* multiple signatures, including required/optional mixes

---

## 10. Open questions (to finalize)

* Do we require `expires_at` / validity windows in core, or keep as policy-only?
* Do we require a specific `key_id` canonical format (fingerprint vs string)?
* Do we standardize a transparency log / append-only ledger for published artifacts (likely a profile)?
* Do we introduce an optional profile to bind signatures to additional derived integrity surfaces (e.g., `manifest_hash`, `pack_sha256`) beyond content identity?