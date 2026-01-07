# Profile: Validation (ShEx)

## Status
Optional standardized profile (Kristal v3)

## Purpose

This profile defines how a Kristal v3 implementation MAY publish **ShEx (Shape Expressions)** artifacts derived from validated Kristal Exchange data to support:
- structural conformance checking by external tooling
- implementer guidance and ecosystem interoperability
- debugging and validation transparency

ShEx outputs are **validation aids**. They do not change the Exchange/Runtime Pack schemas and do not affect content-addressed IDs unless explicitly included in hashed material by a separate profile.

## Scope

This profile specifies:
- required/optional ShEx artifacts
- deterministic generation rules (if artifacts claim determinism)
- packaging and manifest references
- verification and failure semantics when ShEx artifacts are declared

This profile does **not**:
- mandate ShEx as the primary validation mechanism (Kristal validation remains deterministic and schema/rule-driven)
- require ShEx for v3 core conformance
- require specific ShEx engines or toolchains

## Conformance

An implementation claims this profile by including `profile-validation-shex@1` in:
- the Exchange Manifest `profiles[]` (if ShEx is attached to Exchange), and/or
- the Runtime Pack Manifest `profiles[]` (if ShEx is shipped with Runtime Packs)

If an implementation claims this profile, it MUST meet the requirements below.

## Artifacts

### Required artifacts

If this profile is claimed, the implementation MUST provide:

1. **Primary ShEx schema**
   - file: `validation/shex/kristal.shex` (recommended path; any path allowed if referenced in manifest)
   - format: ShExC (`text/shex`) or ShExJ (`application/json`)
   - MUST describe the shapes relevant to the Exchange projection covered by this profile

2. **ShEx metadata descriptor**
   - file: `validation/shex/manifest.json`
   - MUST declare:
     - schema format (ShExC or ShExJ)
     - covered export projection(s)
     - version of the profile (`profile-validation-shex@1`)
     - generation tool info and config hash

### Optional artifacts
- `validation/shex/examples/` (minimal passing/failing examples)
- `validation/shex/mapping.md` (human-readable mapping from Kristal constructs to shapes)
- `validation/shex/reports/` (example validation reports; not normative)

## Coverage and projection rules

ShEx shapes MUST be defined against a **declared RDF projection** of the Exchange. Implementations MUST specify one of:

- **Projection A (WDQS-aligned RDF)**: the RDF projection defined in `05-profiles/profile-rdf-wdqs-export.md`
- **Projection B (JSON-LD RDF mapping)**: the JSON-LD export defined in `05-profiles/profile-jsonld-export.md`, interpreted as RDF
- **Projection C (implementation-defined)**: allowed only if the exact projection is specified in `validation/shex/manifest.json`

The ShEx schema MUST explicitly name which projection(s) it covers.

## Deterministic generation requirements

If the implementation claims deterministic generation for the ShEx artifacts, then:

1. The ShEx schema generation MUST be deterministic given:
   - the same Exchange snapshot
   - the same generation tool version
   - the same generation configuration (captured by a config hash)

2. The ShEx metadata descriptor MUST include:
   - `generator.name`
   - `generator.version`
   - `generator.config_hash` (sha256 hex)
   - `exchange_ref.exchange_id` (sha256 hex)

3. Any ordering within ShExJ output that affects bytes MUST be deterministic (e.g., stable sorting of shapes and predicates).

If deterministic generation is not claimed, the profile MAY still be used, but the metadata MUST set `deterministic=false` and MUST NOT be used as a basis for content-addressed IDs.

## Packaging and manifest references

### If attached to Exchange
- The Exchange Manifest MUST list the ShEx files in its file inventory section (if present), including per-file `sha256`.

### If attached to Runtime Packs
- The Runtime Pack Manifest MUST include the ShEx files in `files[]` with:
  - `role = "metadata"` (or a dedicated role if standardized later)
  - `sha256` and `size_bytes`

## Verification and failure semantics

- If ShEx artifacts are declared in a manifest (Exchange or Runtime Pack) and the consumer is configured to verify package integrity, then:
  - missing ShEx files or hash mismatches MUST cause verification failure (fail-closed)
- ShEx validation execution (running a ShEx engine) is OPTIONAL for consumers and MUST NOT be required for v3 core conformance.

## Minimal `validation/shex/manifest.json` schema (non-normative)

Recommended fields:

```json
{
  "profile": "profile-validation-shex@1",
  "deterministic": true,
  "exchange_ref": { "exchange_id": "..." },
  "projection": {
    "id": "profile-rdf-wdqs-export@1",
    "notes": "Shapes target WDQS-aligned RDF projection."
  },
  "schema": {
    "path": "validation/shex/kristal.shex",
    "format": "ShExC"
  },
  "generator": {
    "name": "kristal-shex-gen",
    "version": "1.0.0",
    "config_hash": "..."
  }
}
