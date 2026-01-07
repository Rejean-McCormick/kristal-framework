# JSON-LD export profile (v3)

## Status

Draft (v3 profile)

## Purpose

This document defines the **JSON-LD 1.1 export profile** for Kristal v3.

Goals:

* Provide a **deterministic**, **portable**, and **Wikibase-aligned** JSON-LD representation of Kristal Exchange content.
* Ensure exporters across languages/toolchains produce **byte-stable output** given the same Exchange content and export policy.
* Define clear boundaries between:

  * Kristal Exchange (canonical JSON source of truth), and
  * JSON-LD export (derived projection for interoperability).

This profile is optional, but if enabled, it is conformance-testable.

---

## 1. Scope and non-goals

### 1.1 In scope

* Deterministic JSON-LD 1.1 structure for exporting claims/statements from Exchange.
* Stable `@context` and `@type` usage to enable consistent RDF interpretation.
* Deterministic ordering rules so the export is byte-stable.
* Rules for encoding:

  * entities (QIDs)
  * properties (PIDs)
  * literals (values, datatypes, language tags)
  * qualifiers
  * references/evidence pointers
  * ranks / truthy mapping (as a projection option)

### 1.2 Out of scope

* Defining the Exchange schema itself.
* Runtime Pack query behavior.
* RDF dataset canonicalization / RDF hashing (handled by the RDF integrity profile).

---

## 2. Profile identification (mandatory)

If JSON-LD export is produced, the exporter MUST declare:

* `export_profile_id = "kristal.v3:jsonld-1.1"`
* `export_profile_version = "1"`

The export MUST also record:

* the `kristal_id` (or content_id) of the Exchange source used
* the `canonicalization_profile` used by Exchange (for auditability)

---

## 3. Determinism requirements (mandatory)

### 3.1 Byte-stable export

Given identical Exchange content and export settings, two implementations MUST output identical JSON bytes after applying:

* the canonical JSON serialization rules used for exports (recommended: JCS for export artifacts)
* the deterministic ordering rules in this profile

### 3.2 Ordering rules

Where arrays represent conceptually unordered sets, they MUST be sorted deterministically.

Minimum required ordering:

* Entities: by entity id (lexicographic)
* Statements: by `statement_id` if present; otherwise by `(subject, property, value, qualifiers_hash, references_hash)`
* Qualifiers: by `(property, value)` lexicographic
* References / evidence pointers: by stable `evidence_id` (or equivalent) lexicographic
* Language maps: by language code lexicographic

No field may be emitted in an implementation-dependent order.

### 3.3 Normalization rules

Exporters MUST preserve normalized values produced by validation/resolution:

* do not re-normalize numbers/dates in export
* do not coerce datatypes beyond what is declared in Exchange
* do not “fix” unresolved ambiguity by picking a best guess

---

## 4. JSON-LD structure (normative)

### 4.1 Top-level document shape

The export MUST be a JSON-LD document with:

* `@context`
* one of:

  * `@graph` (recommended), or
  * top-level node array

The exporter MUST choose one shape and keep it stable within `export_profile_version`.

Recommended:

```json
{
  "@context": { ... },
  "@graph": [ ... ]
}
```

### 4.2 Context requirements (mandatory)

The `@context` MUST:

* be stable and versioned
* define compact terms for:

  * entity identifiers
  * property identifiers
  * statement model fields (qualifiers, references, rank, provenance pointers)

Recommended context strategy:

* Use a single canonical context URI per version (e.g., `https://…/kristal/v3/jsonld/context/v1`)
* Include explicit prefix mappings (e.g., `wd:`, `wdt:`, `p:`, `ps:`, `pq:`, `pr:`) if you align directly to Wikidata-style IRIs.

### 4.3 Node identity

Entity nodes MUST have an `@id` that is stable and derived from the entity identifier.

Recommended:

* QID `Q123` maps to `wd:Q123` (or a declared base IRI + `Q123`)

---

## 5. Statement model (Wikibase-aligned)

### 5.1 Representing claims/statements

The export MUST represent statements in a way that can be interpreted as Wikibase-like statements:

For each subject entity:

* properties map to statements
* each statement includes:

  * main value
  * qualifiers
  * references/evidence pointers
  * rank (if available)

Two acceptable approaches:

1. **Direct Wikidata JSON-LD alignment** (use Wikidata IRIs and statement reification patterns)
2. **Kristal-native JSON-LD** with explicit mappings to Wikibase concepts

The exporter MUST choose one approach and declare it in metadata:

* `statement_model = "wikibase-aligned"` or `statement_model = "kristal-native"`

### 5.2 statement_id inclusion

If `statement_id` exists in Exchange, it MUST be included in export and used for determinism.

If it does not exist, exporters MUST compute a deterministic surrogate key for ordering (not as a new canonical ID unless declared by profile).

---

## 6. Literals and datatypes (mandatory)

### 6.1 Typed literals

Typed values MUST be expressed using JSON-LD value objects:

```json
{ "@value": "2020-01-01", "@type": "xsd:date" }
```

### 6.2 Language-tagged strings

Language values MUST be expressed using:

```json
{ "@value": "Bonjour", "@language": "fr" }
```

### 6.3 Entity references

Values that are entities MUST be `@id` references, not strings.

---

## 7. Qualifiers and references (mandatory)

### 7.1 Qualifiers

Qualifiers MUST be represented as a set/array that is deterministically sorted.

### 7.2 References / evidence pointers

References MUST include stable pointers back to Exchange evidence objects.

Minimum requirement:

* include a stable `evidence_id` / `reference_id` and (optionally) a `content_id` hash for the referenced evidence object.

Export MUST NOT embed large raw evidence blobs unless explicitly enabled by an additional profile.

---

## 8. Ranks and truthy projection

### 8.1 Full vs truthy exports

This profile supports two projections:

* **full**: export all statements with rank metadata
* **truthy**: export only the selected “best” statements per property according to a deterministic rule

If `truthy` is supported, the exporter MUST:

* define the rule (e.g., preferred over normal; latest by time qualifier; stable tie-breakers)
* keep it deterministic
* declare which projection is used in export metadata:

  * `projection = "full"` or `projection = "truthy"`

If truthy rules are not implemented, exporters MUST only emit `projection="full"`.

---

## 9. Integrity linkage (recommended)

The JSON-LD export SHOULD include:

* `source_kristal_id`
* `export_content_id` (hash of the export artifact itself, if you content-address exports)

If the system uses signatures:

* the JSON-LD export MAY be signed as a derived artifact, but this is outside the core and should follow the v3 signature rules.

---

## 10. Conformance tests (mandatory for implementations claiming this profile)

An implementation claiming `kristal.v3:jsonld-1.1` MUST provide fixtures that validate:

* deterministic output bytes for a fixed Exchange input
* correct ordering of entities/statements/qualifiers/references
* correct encoding of literals, datatypes, and language tags
* correct `@context` versioning and stability
* (if truthy supported) truthy projection correctness and determinism

---

## 11. Open questions

* Do we standardize a single canonical `@context` URI and term set, or allow multiple contexts with declared mapping?
* Do we require Wikidata-style IRIs, or allow Kristal-native IRIs with mapping tables?
* How strict should truthy rules be (single global rule vs per-property rules declared in profile)?
