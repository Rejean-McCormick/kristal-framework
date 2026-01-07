# Profile: RDF WDQS Export (Kristal v3)

## Status
Draft

## Purpose
Define a deterministic RDF export profile for Kristal v3 that is compatible with common Wikidata Query Service (WDQS) expectations and downstream RDF tooling, while keeping Kristal’s core identity and offline runtime constraints intact.

This profile specifies:
- what graphs are exported,
- how statements are represented (including qualifiers and references),
- rank/truthy mapping behavior,
- required determinism rules (byte-stability),
- and conformance expectations.

## Scope
In scope:
- RDF dataset shape and serialization requirements
- statement / qualifier / reference modeling
- rank and “truthy/best-rank” projection rules
- deterministic output requirements

Out of scope:
- RDF dataset hashing / canonicalization (handled by the optional **RDF Integrity (RDFC)** profile)
- live SPARQL service behavior (Kristal Runtime Packs remain offline and constrained)
- full WDQS feature parity (this is an export compatibility profile, not a hosted WDQS clone)

## Terminology
- **Item**: Wikibase entity with QID.
- **Property**: Wikibase property with PID.
- **Statement**: a claim about an item using a property and a value, with optional qualifiers, references, and rank.
- **Truthiness**: a projection selecting best-ranked statements for a given (subject, property).

Normative keywords: MUST, SHOULD, MAY.

## Inputs
- Kristal Exchange (canonical source of truth)
- Export configuration selecting:
  - `export_profile = "rdf-wdqs"`
  - `projection = "full" | "truthy"` (see below)
  - optional language preferences for labels (labels are optional in RDF export unless explicitly enabled)

## Outputs
A deterministic RDF dataset serialized as N-Quads (recommended) or Turtle (allowed with additional determinism constraints).

### Required output artifacts
- `export.rdf.nq` (recommended primary)
- `export.manifest.json` (records profile id, version, projection, and determinism settings)

## Dataset structure

### Graphs
The export MUST be a dataset with at least these named graphs:

1. **Assertion graph** (required)
   - Contains statements and their qualifiers and rank.
2. **Reference graph** (required if references exist)
   - Contains reference nodes and reference details.
3. **Metadata graph** (optional)
   - Contains export metadata (build_id, kristal_id, timestamps, profile selection). This graph MUST NOT affect the Kristal core content hash.

Notes:
- If references are present in the source data, a reference graph MUST be emitted.
- If a metadata graph is emitted, it MUST be clearly separated and excluded from any content-hash calculations unless explicitly declared by a profile.

## URI policy (deterministic)
The export MUST define deterministic URI construction for:
- items (QIDs)
- properties (PIDs)
- statement nodes
- reference nodes
- value nodes (where needed)

### Recommended URI scheme (Wikibase-aligned)
- Items: `wd:Q123`
- Direct properties: `wdt:P123`
- Statement properties: `p:P123`
- Statement node: deterministic IRI derived from a stable statement id
- Qualifier properties: `pq:P123`
- Reference properties: `pr:P123`

An implementation MAY use the canonical Wikidata namespace forms, but MUST be consistent and deterministic.

### Statement node identity (critical)
Each exported statement MUST have a stable identifier across runs given identical inputs.

Requirement:
- If the Exchange provides a stable `claim_id`, the statement node SHOULD be derived from it.
- If not, the statement node MUST be derived from a deterministic hash of:
  - subject QID
  - predicate PID
  - normalized object value
  - normalized qualifiers (sorted deterministically)
  - and (optionally) a disambiguator when multiple identical statements exist.

The exact construction MUST be documented and recorded in the export manifest.

## Statement modeling

### Full projection (`projection = "full"`)
The “full” projection MUST emit:
- direct-value triples (`wdt:Pxx`) for the value
- statement node triples (`p:Pxx`) to a statement node
- qualifier triples (`pq:Pxx`) attached to the statement node
- reference triples (`prov:wasDerivedFrom` or Wikibase-style reference linkage) attached to the statement node
- rank representation (see Rank section)

### Truthy projection (`projection = "truthy"`)
The “truthy” projection MUST emit only best-ranked statements per (subject, property) according to rank rules below.

Rules:
- If preferred-rank statements exist for a given (subject, property), only preferred statements are emitted.
- Else, normal-rank statements are emitted.
- Deprecated statements MUST NOT be emitted in truthy projection.
- If multiple statements share the best rank, all best-ranked statements MUST be emitted.

The truthy projection MUST be deterministic given identical inputs.

## Rank rules
Kristal v3’s WDQS export MUST support at least the following ranks:
- `preferred`
- `normal`
- `deprecated`

Rank MUST be represented in a deterministic manner. Acceptable methods include:
- Wikibase-style rank predicates (preferred)
- explicit rank triples using a dedicated predicate in a documented namespace

The method MUST be stated in the export manifest.

## Value modeling
Object values MUST be serialized in a way compatible with WDQS expectations:

- **Item values**: object is another QID node.
- **Strings**: RDF literal, with language tag only where the value is explicitly monolingual.
- **Monolingual text**: literal with language tag.
- **Time**: xsd:dateTime / xsd:date with documented precision handling.
- **Quantity**: numeric literal with unit modeling documented.
- **Coordinates**: WKT literal or dedicated lat/long predicates (documented).
- **URLs**: IRI.

Normalization:
- Values MUST use Kristal’s normalized forms from Exchange (or a documented export normalization step).
- Export normalization must be deterministic and recorded in the export manifest.

## Qualifiers and references
- Qualifiers MUST be emitted as triples attached to the statement node.
- References MUST be emitted as reference nodes; statement nodes MUST link to reference nodes deterministically.

If a statement has multiple references, all MUST be emitted.

## Determinism requirements (mandatory)
The export MUST be byte-stable given identical inputs and the same profile settings.

Minimum requirements:
1. Use a deterministic serializer configuration.
2. Output N-Quads sorted lexicographically by:
   1) graph IRI
   2) subject
   3) predicate
   4) object
3. Ensure stable blank node handling:
   - either avoid blank nodes (preferred), or
   - deterministically skolemize them.

The export manifest MUST record:
- profile id + version
- projection (`full` or `truthy`)
- namespace choices
- statement node identity scheme
- sorting / serialization parameters

## Conformance
An export is conformant to this profile if:
- it emits the required graphs and required modeling for the chosen projection,
- it follows the deterministic URI and statement identity rules,
- it is byte-stable under repeated builds,
- and it records all required parameters in the manifest.

## Recommended tests
- Fixture dataset with known expected N-Quads output.
- Determinism test: same inputs → identical bytes across runs.
- Rank/truthy tests: preferred overrides normal; deprecated excluded from truthy.
- Multi-reference tests: stable ordering of reference emission.

## Interactions with other profiles
- **RDF Integrity (RDFC)** profile (optional): adds `rdf_hash` computation and RDFC CI gating. This WDQS export profile is a prerequisite input for that profile when enabled.
- **Provenance (Nanopub + PROV-O)** profile (optional): may reuse the same statement and reference nodes, but packaging is separate from this export.
