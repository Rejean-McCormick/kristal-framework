
# Validate a Version (Release & Publishing Rules)

## Status
DRAFT

## 1. Goal

This guide defines how to validate and publish a Kristal version so that:
- IDs remain stable and deterministic,
- consumers can trust compatibility guarantees,
- updates can be merged safely,
- registries can serve correct versions and slices.

It covers:
- version types (format vs content),
- compatibility rules,
- validation gates for publication,
- recommended release metadata.

---

## 2. Two Kinds of Versioning

### 2.1 `format_version` (schema/version of the spec)
- Stored in the Kristal as `format_version`
- Indicates compatibility with Kristal format specification
- Changes only when the spec changes (schema-breaking or schema-extending)

Rules:
- Patch: clarifications only (no structural change)
- Minor: backward-compatible additions
- Major: breaking structural changes

### 2.2 Content version (release version of the Kristal pack)
A Kristal pack SHOULD also have a content version:
- stored in manifest as `content_version`

Example:
```json
{
  "manifest": {
    "content_version": "2025.12.19-1",
    "release_channel": "stable"
  }
}
````

This version changes whenever content changes (new claims, edits, merges).

---

## 3. Compatibility Contract

### 3.1 What MUST remain stable within the same format_version

* core field names and required fields
* identifier semantics (Q/P/R/S)
* value typing rules
* canonicalization rules used for hashing

### 3.2 What MAY evolve without breaking consumers

* new optional fields in manifest
* new optional indexes
* new contexts (if default behavior stays safe)
* additional charter entries (new properties)

### 3.3 Breaking changes (require major bump)

* removing/renaming required fields
* changing canonicalization rules used for `kristal_id` or `claim_id`
* changing meaning of existing fields
* changing claim identity construction rules

---

## 4. Validation Gates for Publishing

A Kristal version is publishable if it passes:

### Gate A — Schema & Required Fields

* valid JSON
* required top-level keys exist
* required manifest fields exist

### Gate B — Charter Availability

* all properties used in claims are defined in available charters
* referenced charters are included in pack or resolvable by registry policy

### Gate C — Referential Integrity

* claim subjects exist in `qitems[]` (unless external references allowed)
* QID objects referenced exist or are permitted as external

### Gate D — Value Type Compliance

* claim value types match charter constraints (if defined)
* invalid types are either rejected or downgraded to `needs_review` by policy

### Gate E — Deterministic Ordering

* arrays with set semantics (qitems/claims/charter lists) are deterministically ordered

### Gate F — ID Consistency (if enforced)

* `claim_id` matches canonical hash
* `kristal_id` matches canonical hash

### Gate G — Query Smoke Tests

* basic triple-pattern queries return consistent results without errors
* slicing produces a valid derived Kristal (optional but recommended)

---

## 5. Release Channels (Recommended)

Kristals should declare a channel:

* `stable`:

  * only asserted core claims above confidence threshold
  * strict validation, evidence required by policy

* `beta`:

  * may include inferred claims
  * may include warnings / partial validation

* `dev`:

  * can include placeholders and experimental properties
  * not guaranteed for production use

Example:

```json
{
  "manifest": {
    "release_channel": "stable",
    "validation": { "status": "passed" }
  }
}
```

---

## 6. Lineage Metadata (Required for Derived Versions)

If the Kristal is derived from others (slice/merge), it SHOULD include:

```json
{
  "manifest": {
    "derived_from": [
      { "kristal_id": "kristal://sha256:...", "slice_id": "slice://sha256:..." }
    ]
  }
}
```

This supports audit, rollback, and provenance tracking.

---

## 7. Deterministic Rebuild Rule

A release must be reproducible:

* the same inputs + the same charters + the same policy config
* must produce the same `kristal_id` and `claim_id`s

Non-deterministic steps (LLM decisions) must be captured in trace logs and optionally frozen.

---

## 8. Publishing Checklist (Minimal)

Before publishing:

* [ ] Gate A passes
* [ ] Gate B passes
* [ ] Gate C passes
* [ ] Gate D passes (or warnings accepted)
* [ ] Gate E passes
* [ ] Gate F passes (if hashing enforced)
* [ ] Gate G passes
* [ ] manifest has `content_version`, `release_channel`, and `validation.checked_at`

---

## 9. Recommended Files in a Release

```
dist/
  <name>.kristal.json
  <name>.index.json            (optional)
  <name>.trace.bundle.json     (optional)
  <name>.report.json           (recommended)
```

`<name>.report.json` should include:

* counts of entities/claims
* validator warnings/errors summary
* build inputs (charter ids, policy id)
* hash values (kristal_id, trace_root)


