
# Kristal Format Specification (Enhanced Wikidata / Orgo-Aligned)

## Status
DRAFT

## 1. Scope

This document defines the **Kristal core format**: a portable, offline-friendly knowledge artifact that stays as close as possible to the Wikidata/Wikibase model while supporting Orgo’s enhancements to the property system.

A Kristal is designed to be:
- **downloadable** as a domain-scoped knowledge pack,
- **queryable locally** (like a database),
- **editable and mergeable** (offline-first workflow),
- **auditable** via detached trace logs (hash-referenced).

This document specifies:
- identifiers (Q/P/R/S),
- file structure,
- entity and claim representation,
- trace references,
- charter (property catalog) embedding.

This document does **not** specify:
- query language and slicing rules (see `sparql_queries.md`),
- trace log payloads (see Trace Log spec),
- merge protocols (see merge spec).

---

## 2. Identifier Model (Wikidata + Orgo)

Kristals use a multi-namespace identifier model:

### 2.1 Q — Entities (Wikidata-aligned)
- Format: `Q<digits>` (e.g., `Q42`)
- Entities represent items/concepts/nodes.
- Kristals may include only a domain subset of Q-items to support offline use.

### 2.2 P — Properties (Wikidata-aligned)
- Format: `P<digits>` (e.g., `P31`)
- Use PIDs when semantics match Wikidata.

### 2.3 R — Refinements of P (Orgo extension)
- Format: `R####NNN` (e.g., `R710002`)
- `R####NNN` refines base property `P####` while keeping numeric alignment.
- `NNN` is 001–999.

### 2.4 S — Orgo-specific properties (Orgo extension)
- Format: `S####NNN` or `S0000NNN`
- If `based_on = P####`, then `S####NNN`.
- If `based_on = null` (no Wikidata equivalent), then `S0000NNN`.

> P/R/S catalogs are provided by Orgo “charters” (property charts), optionally embedded in a Kristal.

---

## 3. File Format

A Kristal is a JSON document:

```json
{
  "kristal_id": "kristal://sha256:<hash>",
  "format_version": "1.0.0",
  "manifest": { },
  "charters": { },
  "qitems": [ ],
  "claims": [ ],
  "index": { }
}
````

### 3.1 `kristal_id`

* Content-addressed identifier
* `kristal_id = sha256(canonical_json_without_signature)`
* Canonicalization rules must be stable and deterministic.

### 3.2 `format_version`

Semantic version of the Kristal format itself (not the content version).

---

## 4. Manifest

The manifest provides discovery metadata and trust signals.

```json
{
  "title": "Plants (Core)",
  "description": "Domain Kristal for botany",
  "domains": [
    { "scheme": "unesco", "code": "TBD" },
    { "scheme": "wikidata", "qid": "Q42231" }
  ],
  "created_at": "2025-12-19T00:00:00Z",
  "created_by": "repo:rejean-mccormick/kristal-framework",
  "license": "proprietary",
  "validation": {
    "status": "passed | partial | failed",
    "policy_id": "policy://sha256:<hash>",
    "checked_at": "2025-12-19T00:00:00Z"
  },
  "trace_root": "trace://sha256:<hash>",
  "signature": null
}
```

### 4.1 Required fields

* `title`
* `created_at`
* `created_by`
* `license`
* `validation.status`

### 4.2 Recommended fields

* `domains[]` (UNESCO and/or Wikidata QIDs)
* `trace_root` (for auditability)
* `policy_id`

---

## 5. Charters (P/R/S Catalogs)

Kristals may embed or reference Orgo charters.

```json
{
  "embedded": [
    {
      "name": "general",
      "content": {
        "P": [ { "id": "P31", "label_en": "instance of" } ],
        "R": [ { "id": "R710002", "based_on": "P710", "label_en": "assigned to" } ],
        "S": [ { "id": "S000001", "based_on": null, "label_en": "workflow state" } ]
      }
    }
  ],
  "refs": [
    "charter://sha256:<hash>"
  ]
}
```

Rules:

* A Kristal MUST be able to compile/validate using only embedded/available charter definitions.
* When using `refs`, resolution requires fetching referenced charts (registry behavior is defined elsewhere).

---

## 6. Q-Items (Entities)

Kristals store a bounded set of entities for offline querying.

```json
{
  "id": "Q123",
  "labels": { "en": "Photosynthesis", "fr": "Photosynthèse" },
  "descriptions": { "en": "Biological process..." },
  "aliases": { "en": ["photosynthetic process"] }
}
```

Notes:

* Entities may include only the fields needed for local use.
* Richer entity payloads are allowed (e.g., extra label languages).

---

## 7. Claims

Kristals store claims as a **flat list** to make diffs and merges simple.

### 7.1 Claim object

```json
{
  "claim_id": "claim://sha256:<hash>",
  "subject": "Q123",
  "property": "P31",
  "value": { "type": "wikibase-entityid", "content": "Q33742" },
  "qualifiers": [],
  "references": [],
  "confidence": 0.97,
  "status": "asserted | inferred | conflicted | rejected | needs_review",
  "trace_refs": ["trace://sha256:<hash>"]
}
```

### 7.2 Property namespace

`property` may be:

* `P<digits>`
* `R####NNN`
* `S####NNN`
* `S0000NNN`

### 7.3 Value types

Minimum supported types:

* `wikibase-entityid` (QID)
* `string`
* `monolingualtext`
* `time`
* `quantity`
* `globecoordinate`
* `url`

Example (time):

```json
{ "type": "time", "content": { "iso": "2024-01-31", "precision": "day" } }
```

### 7.4 Qualifiers and references

Qualifiers and references are modeled as arrays of property-value pairs:

```json
{
  "qualifiers": [
    { "property": "P580", "value": { "type": "time", "content": { "iso": "2020-01-01", "precision": "day" } } }
  ],
  "references": [
    { "source_ref": "source://sha256:<hash>", "locator": { "url": "https://example.org", "offset": [120, 220] } }
  ]
}
```

---

## 8. Trace References (Detached Logs)

Kristals do not embed heavy provenance by default.
Instead, they reference external trace logs:

* `manifest.trace_root`
* `claim.trace_refs[]`
* `reference.source_ref`

Trace payloads are defined in the Trace Log specification.

---

## 9. Index (Optional)

Kristals may include an `index` block to support offline resolution and faster local queries:

```json
{
  "labels": {
    "photosynthesis": ["Q123"]
  },
  "aliases": {
    "photosynthetic process": ["Q123"]
  }
}
```

Guidance:

* The index is optional.
* The index MUST be reproducible from `qitems` (and optional extra sources) to avoid becoming a source of truth.

---

## 10. Minimal Valid Kristal (Example)

```json
{
  "kristal_id": "kristal://sha256:...",
  "format_version": "1.0.0",
  "manifest": {
    "title": "Plants (Core)",
    "created_at": "2025-12-19T00:00:00Z",
    "created_by": "repo:rejean-mccormick/kristal-framework",
    "license": "proprietary",
    "validation": { "status": "partial" }
  },
  "charters": { "embedded": [], "refs": [] },
  "qitems": [
    { "id": "Q123", "labels": { "en": "Photosynthesis" }, "descriptions": { "en": "..." }, "aliases": {} }
  ],
  "claims": [
    {
      "claim_id": "claim://sha256:...",
      "subject": "Q123",
      "property": "P31",
      "value": { "type": "wikibase-entityid", "content": "Q33742" },
      "qualifiers": [],
      "references": [],
      "confidence": 0.97,
      "status": "asserted",
      "trace_refs": []
    }
  ],
  "index": {}
}
```

---

## 11. Conformance

An implementation conforms to this spec if it can:

* parse and validate the Kristal JSON structure,
* interpret Q/P/R/S identifiers,
* enforce value typing and charter constraints,
* execute local queries over `qitems` + `claims`,
* preserve trace references without requiring embedded logs.


