Voici **8 / 18** — le contenu proposé pour **`docs/4-kristal-exchange.md`**.

---

# 4. Kristal Exchange Format (v0.1)

## 4.1 Purpose

**Kristal Exchange** is the canonical, shareable representation of a Kristal.

It is:

* **Wikibase-aligned** (statements, qualifiers, references)
* suitable for distribution and archival
* convertible to Wikibase JSON and RDF/JSON-LD

It is **not** optimized for execution.
Execution is handled by the **Kristal Runtime Pack**.

---

## 4.2 Design constraints

Kristal Exchange must:

1. Preserve Wikidata semantics and identifiers
2. Carry evidence and provenance
3. Support partial/unknown resolution without data loss
4. Be stable for long-term reuse
5. Be compilable into multiple runtime formats

---

## 4.3 Top-level structure

A Kristal Exchange document is a single JSON object:

* `schema_version`
* `kristal_id` (content-addressed)
* `manifest` (metadata)
* `entities[]` (usually one primary entity)
* `unresolved[]` (optional)
* `evidence_pack_ref` (optional external linkage)

**Primary assumption (v0.1):** one primary entity per Kristal.
Multi-entity Kristals are allowed but discouraged in v0.1.

---

## 4.4 Manifest

The `manifest` carries:

* title / domain tags
* creation metadata
* validation status
* provenance summaries
* signatures (optional)

Recommended fields:

* `title`
* `domain[]`
* `created_at`
* `created_by`
* `source_summary[]`
* `validation_status` ∈ { `passed`, `partial`, `failed` }
* `confidence_summary` (optional)
* `signatures[]` (optional)

The manifest is intended for:

* registries
* catalogs
* offline pack management

---

## 4.5 Entities

Each entity is Wikibase-like:

* `id` (QID or local placeholder)
* `labels` (lang → string)
* `descriptions` (lang → string)
* `aliases` (lang → [string])
* `claims` (property → statement[])

Example (conceptual):

```json
{
  "id": "Q7259",
  "labels": { "en": "Ada Lovelace" },
  "claims": {
    "P569": [ ... ],
    "P27":  [ ... ]
  }
}
```

---

## 4.6 Statements

A statement is an object containing:

* `rank` (optional)
* `mainsnak` (property + datavalue)
* `qualifiers` (optional)
* `references` (optional)
* `statement_id` (optional but recommended)

This is intentionally close to Wikibase JSON, but slightly simplified.

### 4.6.1 Snaks and datavalues

A `mainsnak` contains:

* `property` (PID string)
* `datatype` (optional)
* `datavalue` (typed)

Datavalue types align with Wikibase:

* item values (QIDs)
* time values
* quantities
* monolingual text
* coordinates
* strings, URLs

---

## 4.7 References (provenance)

References are first-class:

* A statement may include multiple references
* Each reference may include multiple snaks

Minimal recommended reference fields:

* `P854` (reference URL)
* `P813` (retrieved date)
* optional `quote` and `offsets` via extended fields (non-Wikibase but exportable)

Kristal Exchange supports two provenance modes:

### Inline provenance

References are embedded inside the statement (Wikibase-style).

### Externalized provenance (recommended for scale)

The statement includes only a reference pointer (`evidence_id`) and the detailed evidence lives in a separate Evidence Pack.

This keeps the Exchange file manageable while preserving traceability.

---

## 4.8 Unresolved and partial content

Kristal Exchange is allowed to contain claims that are not fully resolved, but they must be explicit.

`unresolved[]` may contain:

* unresolved subject candidates
* unresolved property candidates
* unresolved item candidates
* unresolved literal parsing

Rules:

* Unresolved content is never silently dropped
* It must not be exported as authoritative Wikibase claims unless resolved
* It may still be retained for human review or later compilation

---

## 4.9 Identity and hashing

Kristal Exchange uses content-addressing:

* `kristal_id = sha256(canonical_json(exchange_document_without_signatures))`

Statement identity:

* `statement_id` may be computed as a hash of canonical (subject, pid, value, qualifiers, reference pointers)

This supports:

* deduplication
* merge operations
* distributed caching
* tamper evidence

---

## 4.10 Backward compatibility and export

Kristal Exchange must support:

1. Export to **Wikibase JSON** (lossless for supported features)
2. Export to **RDF / JSON-LD**
3. Degradation: if some fields cannot be exported, they must be preserved in `unresolved[]` or extensions, not discarded

Kristal Exchange does not require RDF internally, but supports it as an output.

---

## 4.11 Summary

Kristal Exchange is the stable, shareable artifact:

* aligned with Wikibase semantics
* provenance-aware
* merge-friendly
* suitable for offline distribution

Execution speed is delegated to Runtime Packs.

---

**Next document:** `4-kristal-exchange.schema.json`
