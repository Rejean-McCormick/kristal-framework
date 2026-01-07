````md
# JCS Test Vectors (Kristal v3)

## Status
Draft (normative test-vector guidance)

## Purpose
This folder contains **golden test vectors** for Kristal v3’s canonicalization and identity rules.

Kristal v3 requires that:
- `canonical_json` is **RFC 8785 JSON Canonicalization Scheme (JCS)**, and
- content-addressed IDs (e.g., `kristal_id`) are computed as:

`sha256( JCS(json_object_without_signatures) )`

These vectors ensure that independent implementations in different languages compute identical canonical bytes and identical hashes.

---

# Files in this folder

- `vectors.json`  
  A collection of JCS input cases and expected canonical outputs.

- `expected-hashes.txt`  
  Expected SHA-256 hashes for each vector’s canonical output.

---

# Vector format (recommended)

Each entry in `vectors.json` SHOULD use the following shape:

```json
{
  "id": "jcs-0001",
  "description": "Brief description of the case",
  "input": { "...": "original JSON object" },
  "expected_canonical": "{\"a\":1,\"b\":2}",
  "expected_sha256_hex": "..."
}
````

Notes:

* `expected_canonical` MUST be the exact UTF-8 string output of JCS.
* `expected_sha256_hex` MUST be the SHA-256 digest of `expected_canonical` bytes (UTF-8), lower-case hex.

Implementations MAY split `expected_sha256_hex` into `expected-hashes.txt` instead of embedding it in JSON, but the correspondence MUST be unambiguous.

---

# Required coverage (minimum)

The vector set MUST include cases covering:

## 1) Object key ordering

* keys out of order in input
* nested objects

## 2) Arrays

* arrays preserve order
* nested arrays/objects

## 3) Numbers (critical)

* integers vs floating-point
* negative values
* edge cases around exponent notation
* values that must not be represented in a non-canonical form

## 4) Strings and escaping

* quotes, backslashes
* unicode escapes
* control characters

## 5) Booleans and null

* `true`, `false`, `null`

## 6) “Signature exclusion” fixture (Kristal-specific)

Include at least one fixture where:

* the input contains a signature envelope section (or mock signature fields),
* the canonicalization/hash input explicitly excludes those fields,
* the expected canonical output corresponds to “without signatures.”

This ensures implementations match Kristal’s “remove signatures → canonicalize → hash” rule.

---

# How to use these vectors

## Step 1: Canonicalization

For each vector:

1. Parse `input` into an in-memory JSON object.
2. Serialize it using **JCS**.
3. Compare the resulting bytes (UTF-8) to `expected_canonical`.

## Step 2: Hashing

1. Compute SHA-256 over the canonical bytes.
2. Compare to `expected_sha256_hex` (or the corresponding line in `expected-hashes.txt`).

Any mismatch indicates:

* the implementation is not RFC 8785 compliant, or
* the implementation is applying additional transformations, or
* signature fields were not excluded correctly.

---

# Conformance requirement

A Kristal v3 core implementation MUST:

* ship a vector set (this folder), and
* pass all vectors in CI.

If an implementation claims conformance but fails these vectors, its IDs are not interoperable.

---

# Updating the vectors

## Versioning

If vectors change materially:

* increment the vector set version (add `vector_set_version` to `vectors.json` root or to this README),
* preserve old vectors if possible to avoid breaking downstream CI.

## Adding new vectors

Prefer adding new IDs rather than modifying existing ones, unless correcting an error.

---

# Suggested commands (informative)

Examples of what a test runner should do:

* Load `vectors.json`
* For each vector, run:

  * `canonical = JCS(input)`
  * `hash = sha256(canonical)`
  * assert equality with expected values

The exact runner is language-specific.

```
```
