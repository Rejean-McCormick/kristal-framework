# 01-core-spec/signatures-trust.md

## Status

Draft (v3)

## Purpose

This document specifies the **normative** rules for:

* signature and attestation structures used in Kristal v3 artifacts
* what is signed (and how signing payloads are constructed)
* verification requirements, including **fail-closed** semantics
* trust-root, key identity (`kid`), and offline verification expectations
* minimum interoperability requirements (mandatory-to-implement algorithms)

This document applies to:

* **Kristal Exchange** artifacts
* **Runtime Pack** manifests (and, optionally, other derived artifacts)

---

## 1. Terminology

* **Artifact**: a JSON object (Exchange or Runtime Pack manifest).
* **Content hash**: the artifact’s content-addressed hash (e.g., `kristal_id` for Exchange), computed per `01-core-spec/ids-canonicalization-hashing.md`.
* **Signature**: a cryptographic proof that binds an issuer to a signing payload.
* **Attestation**: a signature that may be produced by a third party (auditor, publisher, distributor) over an artifact payload.
* **Trust root**: a pinned set of public keys (or certificates) trusted by a verifier for a given tenant/environment.
* **Fail-closed**: if an artifact declares hashes/signatures, verification failure is a hard error; the artifact MUST NOT be treated as valid.

Normative keywords: MUST, MUST NOT, SHOULD, SHOULD NOT, MAY.

---

## 2. Core principles

1. **Content IDs are not signatures.**
   `kristal_id` proves content identity; signatures prove issuer authenticity/authorization.

2. **Signatures MUST NOT be included in the content-hash target.**
   Signatures are overlays; the content hash is computed over the artifact with signature fields excluded (see IDs/canonicalization spec).

3. **Verification must be offline-capable.**
   A verifier must be able to verify a signed artifact with no network access, given the artifact and the relevant trust roots.

4. **If declared, integrity MUST be enforced.**
   Presence of declared signatures/hashes implies fail-closed behavior.

---

## 3. Signature envelope

### 3.1 Where signatures live (normative)

Artifacts that support signatures MUST include a top-level field:

* `signatures`: an array of signature objects

If `signatures` is absent or empty, the artifact is considered **unsigned**.

### 3.2 Signature object schema (normative)

Each entry in `signatures[]` MUST be an object with:

* `kid` (string): key identifier used to locate the issuer public key in the trust store
* `alg` (string): signing algorithm identifier
* `hash_alg` (string): hash algorithm used to hash the signing payload
* `payload` (object): the signing payload (see Section 4)
* `payload_hash` (string): `"<hash_alg>:<hex>"` of the canonical payload (see Section 4)
* `sig` (string): signature bytes encoded as base64url (no padding)
* `created` (string): ISO 8601 timestamp
* `required` (boolean, optional): defaults to `true`

Optional fields (profiles / non-normative unless explicitly enabled):

* `issuer` (string): human-readable issuer name
* `expires` (string): ISO 8601 timestamp
* `x5c` (array of strings): X.509 certificate chain (base64 DER) (optional profile)
* `purpose` (string): e.g., `"publisher"`, `"auditor"`, `"distributor"`
* `notes` (string)

### 3.3 Mandatory-to-implement algorithms (interop)

To ensure interoperability, v3 implementations MUST support verification for:

* `alg = "ed25519"`
* `hash_alg = "sha256"`

Implementations MAY support additional algorithms (e.g., ECDSA P-256), but MUST NOT claim v3 conformance if they cannot verify Ed25519 + SHA-256 signatures.

---

## 4. Signing payload construction

### 4.1 Payload goals

The signature payload MUST:

* bind the issuer to the artifact’s **content identity**
* bind the issuer to the canonicalization/hash profile used
* prevent ambiguous “what exactly was signed” situations

### 4.2 Payload fields (normative)

The `payload` object MUST contain:

* `artifact_kind` (string): `"exchange"` or `"runtime_pack_manifest"`
* `artifact_version` (string): the artifact format version (e.g., `"v3"`)
* `content_id` (string):

  * Exchange: the `kristal_id` value
  * Runtime Pack manifest: the pack content-id (if defined) or a deterministic manifest id
* `canonicalization_profile` (string)
* `canonicalization_version` (string)
* `hash_alg` (string): MUST match `signatures[].hash_alg`
* `content_hash` (string): MUST be the hash portion of `content_id` or an equivalent canonical representation

Optional payload fields:

* `tenant_id` (string): tenant scope when applicable
* `environment` (string): e.g., `"prod"`, `"staging"`
* `valid_from` / `valid_until` (ISO 8601 strings)
* `policy_set_id` (string): identifier for the portable policy selections used (if you want signatures to bind to policy choices)

### 4.3 Canonicalization of payload (normative)

To compute `payload_hash`, implementations MUST:

1. canonicalize `payload` using the same canonicalization rules as the artifact (`canonicalization_profile`), i.e., JCS in v3 core
2. compute `payload_hash = sha256(canonical_payload_bytes)`
3. encode as: `payload_hash = "sha256:<lowercase-hex>"`

### 4.4 Signature computation (normative)

For `alg="ed25519"`:

* signature input MUST be the raw bytes of the SHA-256 digest of the canonical payload (32 bytes).
* `sig` MUST be base64url-encoded signature bytes (no padding).

---

## 5. Verification rules (fail-closed)

### 5.1 Verification procedure (normative)

Given an artifact `A`:

1. If `A.signatures` is absent or empty: artifact is **unsigned** (verification MAY succeed with “unsigned” status).
2. Otherwise, for each signature object `S` in `A.signatures`:

   * Validate required fields exist and are well-formed.
   * Recompute canonical payload bytes from `S.payload`.
   * Recompute `payload_hash` and compare to `S.payload_hash`.
   * Resolve the public key for `S.kid` from the verifier’s trust store.
   * Verify `S.sig` under `S.alg` using the verified `payload_hash` bytes.
3. Apply required/optional semantics:

   * If `S.required` is true or missing, it is required.
   * If any required signature fails, verification MUST fail.

### 5.2 Fail-closed semantics (normative)

If an artifact contains any signature object that is **declared** (present in `signatures[]`) and that signature is **required**, then:

* any verification failure (malformed payload, missing key, mismatch, bad signature) MUST be treated as a hard error
* the artifact MUST NOT be accepted for compilation, execution, distribution, or rendering

Unsigned artifacts MAY be accepted depending on deployment policy, but must be clearly treated as unsigned.

### 5.3 Partial verification (normative)

Verifiers MUST NOT “silently skip” signatures they cannot process if those signatures are required.

If a verifier does not support the algorithm referenced by a required signature:

* verification MUST fail (hard error)

---

## 6. Trust roots, key identity, and multi-tenancy

### 6.1 Trust roots (normative)

Each verifier MUST maintain a trust store containing:

* a set of trusted public keys (or certificates) keyed by `kid`

Trust stores MAY be:

* tenant-scoped
* environment-scoped (prod/staging)
* device-scoped (pinned to an app install)

### 6.2 `kid` format (recommended)

Recommended `kid` formats:

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

* **Orgo**: is the control plane enforcing “no compile on validation fail” and signature policies for publish/release.
* **Konnaxion**: must verify signatures before activating packs when signatures are present; must pin trust roots per tenant/environment.
* **Architect**: must only render from validated artifacts; should surface signature/issuer metadata where relevant.

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

* positive verification cases (valid payload_hash + signature)
* negative cases (payload tampering, wrong key, malformed signature, unsupported alg on required signature)
* fail-closed behavior (required signature failure blocks acceptance)

Test vectors SHOULD include:

* Exchange artifacts and Runtime Pack manifests
* multiple `kid` resolutions
* multiple signatures, including required/optional mixes

---

## 10. Open questions (to finalize)

* Do we require `expires` / validity windows in core, or keep as policy-only?
* Do we require a specific `kid` canonical format (fingerprint vs string)?
* Do we standardize a transparency log / append-only ledger for published artifacts (likely a profile)?
