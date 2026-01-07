Voici **16 / 18** — le contenu proposé pour **`docs/8-security-integrity.md`**.

---

# 8. Security, Integrity, and Trust Model (v0.1)

## 8.1 Purpose

This document defines how Kristals ensure:

* **integrity** (content has not been altered),
* **traceability** (who produced what, from which sources),
* **verifiability** (claims can be audited),
* **offline trust** (no reliance on live authorities).

Kristal is **not** a security platform; it provides **cryptographic primitives and conventions** that higher-level systems may use.

---

## 8.2 Threat model (explicit)

Kristal is designed to mitigate:

* accidental corruption of knowledge packs
* silent modification of claims or evidence
* ambiguity about provenance or authorship
* replay or duplication of outdated artifacts

Kristal does **not** attempt to mitigate:

* malicious cryptographic key compromise
* social or editorial disputes
* misinformation at the source level

Truth assessment remains a **human or community process**.

---

## 8.3 Content-addressable identity

### 8.3.1 Kristal identity

Each Kristal Exchange document has a **content-derived identifier**:

```
kristal_id = sha256(canonical_json(exchange_without_signatures))
```

Properties:

* deterministic
* reproducible
* collision-resistant
* tamper-evident

Any modification to the content produces a new ID.

---

### 8.3.2 Statement / claim identity

Optionally, each statement may have a stable identifier:

```
statement_id = hash(subject + property + value + qualifiers + reference_ids)
```

This enables:

* deduplication
* merge operations
* fine-grained provenance tracking

---

## 8.4 Canonicalization rules

To ensure stable hashing, Kristal defines canonicalization rules:

* JSON keys sorted lexicographically
* arrays sorted where order is not semantically meaningful
* normalized time, quantity, and coordinate formats
* whitespace and formatting ignored

Canonicalization is **normative** and must be consistent across implementations.

---

## 8.5 Evidence integrity

Evidence is treated as **referenced material**, not embedded truth.

Rules:

* Evidence objects are immutable once referenced
* Evidence packs may be content-addressed separately
* Statements reference evidence via IDs or structured snaks

Kristal guarantees:

* evidence **cannot be silently swapped**
* missing evidence is explicit and detectable

Kristal does **not** guarantee:

* the authenticity of external URLs
* the permanence of third-party content

---

## 8.6 Signatures and attestations (optional)

Kristal supports **optional cryptographic signatures** over:

* Kristal Exchange documents
* Runtime Pack manifests

Supported signature types (v0.1):

* PGP
* X.509
* Ed25519 (recommended for lightweight/offline use)

A signature asserts:

> “This actor attests to the content of this Kristal as of this hash.”

Signatures do **not** imply correctness, only authorship or endorsement.

---

## 8.7 Trust layers

Kristal separates **technical integrity** from **social trust**.

### Technical (enforced by format)

* hashing
* canonicalization
* schema validation

### Social (out of scope, but enabled)

* trusted signers
* registries
* community review
* reputation systems

This mirrors how systems like Wikidata separate data openness from community governance.

---

## 8.8 Runtime Pack integrity

Runtime Packs are **derived artifacts** and must be verifiable:

* manifest hash covers:

  * dictionary paths
  * triple tables
  * indexes
  * filters
* runtime tools must refuse:

  * missing files
  * hash mismatches
  * incompatible versions

Runtime Packs may be safely discarded and rebuilt from Exchange artifacts.

---

## 8.9 Offline verification workflow

Typical offline verification:

1. Load Kristal Exchange
2. Verify schema validity
3. Recompute `kristal_id`
4. Verify signatures (if present)
5. Verify evidence pack hashes (if present)
6. Optionally rebuild Runtime Pack and compare hashes

No network access is required at any step.

---

## 8.10 Explicit non-goals

Kristal does **not**:

* provide encryption at rest
* manage key distribution
* define trust authorities
* adjudicate conflicting claims

These are deliberately left to higher-level systems or communities.

---

## 8.11 Summary

Kristal security is based on:

* **immutability via hashing**
* **explicit provenance**
* **optional attestations**
* **offline verifiability**

This is sufficient to make Kristals:

* auditable
* reproducible
* mergeable
* suitable for long-term knowledge preservation

---

**Next document:** `9-pipeline.md`
