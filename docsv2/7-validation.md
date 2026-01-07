Voici **15 / 18** — le contenu proposé pour **`docs/7-validation.md`**.

---

# 7. Validation and Conformance (v0.1)

## 7.1 Purpose

This document defines **how Kristals are validated**, **what “valid” means**, and **how partial or degraded content is handled**—from Claim-IR to Kristal Exchange to Runtime Pack.

Validation is **deterministic**, **offline**, and **repeatable**.

---

## 7.2 Validation stages (pipeline)

Validation is staged to avoid mixing concerns:

1. **Claim-IR structural validation**
2. **Resolution validation (QID/PID/value)**
3. **Semantic validation (constraints)**
4. **Kristal Exchange validation**
5. **Runtime Pack validation**

Each stage may produce **errors** or **warnings**. Errors block progression; warnings do not.

---

## 7.3 Claim-IR validation

### 7.3.1 Structural (schema)

* JSON must conform to `claim-ir.schema.json`
* Required fields present
* No additional properties

**Failure → ERROR (blocking)**

### 7.3.2 Minimal compilability

* Each claim has a predicate and object
* Object kind is supported
* Datatype shape is coherent

**Failure → ERROR (blocking)**

### 7.3.3 Evidence profile (optional but recommended)

Depending on the chosen profile:

* **Grounded profile**: at least one evidence per claim required
* **Exploratory profile**: evidence optional

**Failure → WARNING (or ERROR if grounded profile is enforced)**

---

## 7.4 Resolution validation (Sentient / resolver)

Resolution attempts map:

* surfaces → QIDs / PIDs
* literals → typed values

Outcomes:

* **Resolved**: QID/PID assigned
* **Ambiguous**: multiple candidates
* **Unresolved**: surface preserved

Rules:

* Unresolved content is preserved explicitly
* No silent coercion or guessing

**Unresolved → WARNING**
**Type mismatch after resolution → ERROR**

---

## 7.5 Semantic validation (constraints)

Kristal applies a **subset of Wikidata-style constraints** suitable for offline validation.

Examples:

* property datatype compatibility (e.g., P569 expects time)
* value class constraints (e.g., P31 values are items)
* single-value vs multi-value expectations (advisory)

Sources of constraints:

* Wikidata property metadata
* locally cached constraint profiles
* optional ShEx-like rules

**Failure handling**

* Hard constraint violation → ERROR
* Soft constraint violation → WARNING

Kristal does **not** enforce global completeness or community norms.

**Alignment:** constraints mirror those used in Wikidata where feasible, without requiring online access.

---

## 7.6 Kristal Exchange validation

### 7.6.1 Structural

* Must conform to `kristal-exchange.schema.json`
* Required top-level fields present
* Entity/statement shapes valid

**Failure → ERROR**

### 7.6.2 Referential integrity

* QIDs/PIDs referenced are syntactically valid
* Local placeholders are explicitly marked (`local:*`)
* References point to valid evidence entries or packs

**Failure → ERROR**

### 7.6.3 Identity and hashing

* `kristal_id` must match canonical hash of content
* Statement IDs (if present) must be stable under canonicalization

**Failure → ERROR**

---

## 7.7 Runtime Pack validation

Runtime Packs are **derived artifacts** and must satisfy:

* manifest conforms to `kristal-runtime.schema.json`
* dictionaries cover all referenced IDs
* triple counts match statistics
* index paths exist and are readable

Optional checks:

* index encoding compatibility
* filter false-positive rate bounds

**Failure → ERROR (pack unusable)**

---

## 7.8 Validation statuses

Each Kristal Exchange includes a `validation_status`:

* `passed` — all checks passed, no unresolved blocking issues
* `partial` — valid but with unresolved/ambiguous content
* `failed` — structural or semantic errors present

Rules:

* `failed` Kristals must not be compiled into Runtime Packs
* `partial` Kristals may be compiled with unresolved content excluded

---

## 7.9 Error and warning codes (normative)

### Errors (blocking)

* `SCHEMA_INVALID`
* `MISSING_REQUIRED_FIELD`
* `UNSUPPORTED_DATATYPE`
* `TYPE_MISMATCH`
* `INVALID_REFERENCE`
* `HASH_MISMATCH`
* `RUNTIME_INTEGRITY_ERROR`

### Warnings (non-blocking)

* `UNRESOLVED_QID`
* `UNRESOLVED_PID`
* `AMBIGUOUS_ENTITY`
* `SOFT_CONSTRAINT_VIOLATION`
* `EVIDENCE_MISSING`
* `LOW_CONFIDENCE`

Implementations must emit **machine-readable codes** and **human-readable messages**.

---

## 7.10 Repair loop (AI-assisted correction)

When Claim-IR or Exchange validation fails, a **repair loop** may be used:

1. Collect errors/warnings as structured data
2. Provide them to an AI (or human) as constraints
3. Request **minimal corrections only**
4. Re-validate

Rules:

* No new claims may be added during repair unless explicitly requested
* Evidence must not be fabricated
* Each repair attempt is logged

This enables **deterministic AI assistance** without reintroducing hallucinations.

---

## 7.11 Offline guarantees

All validation steps:

* run without network access
* rely on local schemas, dictionaries, and constraint profiles
* produce reproducible results

This is essential for:

* archival
* regulated environments
* reproducible AI pipelines

---

## 7.12 Summary

Validation in Kristal:

* is staged and explicit
* preserves uncertainty instead of hiding it
* aligns with Wikidata constraints where practical
* enables deterministic compilation and execution

Validation is what turns **structured proposals** into **executable knowledge**.

---

**Next document:** `8-security-integrity.md`
