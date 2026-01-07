# Test a Kristal (Before Publishing or Merging)

## Status
DRAFT

## 1. Goal

This guide defines a minimal test suite to ensure a Kristal is:
- structurally valid,
- referentially consistent,
- queryable offline,
- compliant with charters (P/R/S),
- stable for merging (deterministic ordering + IDs).

Testing is split into:
- **static checks** (schema + integrity),
- **semantic checks** (constraints + types),
- **query checks** (basic retrieval works),
- **pack checks** (offline completeness).

---

## 2. Test Levels

### Level 0 — Parse & Schema
**Objective**
- file is valid JSON
- required top-level keys exist
- required manifest fields exist

**Checks**
- JSON parse
- `format_version` exists
- `manifest.title`, `manifest.created_at`, `manifest.created_by`, `manifest.license`, `manifest.validation.status` exist
- `qitems` is an array
- `claims` is an array

**Fail conditions**
- any required field missing
- malformed JSON

---

### Level 1 — Referential Integrity
**Objective**
- claims do not reference missing objects

**Checks**
- every `claim.subject` exists in `qitems[].id` OR is explicitly allowed as external/placeholder by policy
- every QID referenced in `claim.value` (when value is `wikibase-entityid`) exists in `qitems[]` OR is allowed as external
- `claim.property` is a valid ID format (P / R / S)

**Fail conditions**
- missing mandatory Q-items
- invalid property format

---

### Level 2 — Charter Compliance (P/R/S)
**Objective**
- properties used by claims are defined in loaded charters
- value types match property constraints

**Checks**
- load embedded charters or referenced charter files
- confirm every `claim.property` exists in the charter catalog
- confirm `claim.value.type` matches charter `value_type` (if specified)
- check cardinality constraints if defined (single vs multi)

**Fail conditions**
- unknown property ID
- value type mismatch
- constraint violations (policy decides warn vs fail)

---

### Level 3 — Claim Identity Consistency
**Objective**
- ensure `claim_id` matches canonical payload

**Checks**
- recompute claim hash from:
  - subject, property, typed value, qualifiers, reference hashes (as defined by compiler rules)
- confirm stored `claim_id` equals computed

**Fail conditions**
- any mismatch between stored and computed claim_id

---

### Level 4 — Ordering & Determinism
**Objective**
- stable ordering produces reproducible diffs

**Checks**
- `qitems[]` sorted by `id`
- `claims[]` sorted by deterministic keys (subject, property, value canonical string, qualifiers, refs)
- charter lists `P/R/S` sorted by `id`

**Fail conditions**
- non-deterministic ordering in arrays that represent sets

---

### Level 5 — Minimal Query Tests
**Objective**
- ensure the Kristal can be queried like a database

**Required test queries**
1) Outgoing edges:
- retrieve all `(property, object)` for a known subject
2) Incoming edges:
- retrieve all `(subject, property)` for a known object
3) Pattern match:
- retrieve all subjects matching a `(property, value)` pattern
4) Filter:
- retrieve only claims above a confidence threshold and allowed statuses

**Pass criteria**
- query engine returns consistent results
- no crashes on empty results

---

### Level 6 — Slice Test (Offline Subset)
**Objective**
- ensure slicing creates a valid derived Kristal

**Checks**
- run a slice by query and export
- validate the derived output through Levels 0–5
- ensure lineage metadata exists (derived_from)

---

### Level 7 — Pack Completeness (Optional)
**Objective**
- ensure offline pack includes needed dependencies

**Checks**
- if charters are referenced, ensure they are included in pack
- if index is required for offline resolution, ensure it is present
- if traces are included, ensure trace bundle IDs match

---

## 3. Test Output Artifacts

Recommended output files:

```

reports/
test.summary.json
validation.report.json
hash.checks.json
query.smoke.json

````

Example `test.summary.json`:

```json
{
  "kristal_id": "kristal://sha256:...",
  "tests": {
    "schema": "pass",
    "referential_integrity": "pass",
    "charter_compliance": "warn",
    "claim_id_consistency": "pass",
    "ordering": "pass",
    "query_smoke": "pass",
    "slice": "not_run"
  },
  "warnings": [
    { "code": "missing_evidence", "count": 12 }
  ],
  "failed": []
}
````

---

## 4. Recommended CLI (Conceptual)

```bash
kristal validate --in <kristal.json> --out reports/validation.report.json
kristal test --in <kristal.json> --out reports/test.summary.json
kristal query --in <kristal.json> --sparql "<query>" --out reports/query.smoke.json
kristal slice --in <kristal.json> --seed Q123 --closure outgoing --depth 1 --out dist/slice.kristal.json
kristal test --in dist/slice.kristal.json --out reports/slice.test.summary.json
```

---

## 5. Minimum Publish Gate

A Kristal should be publishable only if:

* Levels 0–2 pass
* Level 3 passes if claim_id hashing is enforced
* Level 5 passes (query smoke)

Everything else may be optional depending on policy.


