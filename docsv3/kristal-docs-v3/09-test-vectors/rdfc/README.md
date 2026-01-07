```markdown
# RDFC Test Vectors and Fixtures (Optional Profile)

## Status
Draft

## Purpose
This directory contains pointers and packaging conventions for the **optional** Kristal v3 profile:

- `profile-rdf-integrity-rdfc` (RDF dataset canonicalization + `rdf_hash` + CI gating)

The goal is to make RDF integrity **verifiable and interoperable** by:
- standardizing how fixtures are stored and referenced,
- standardizing how expected hashes are recorded,
- documenting which RDFC conformance tests are gated in CI,
- and ensuring resource limits are captured alongside fixtures.

## Scope
In scope:
- Fixture layout and naming
- Required metadata for reproducing `rdf_hash`
- Expected output recording
- CI gating configuration pointers

Out of scope:
- Shipping the full W3C RDFC test suite inside this repo (implementations may vendor it or reference it)

## Directory layout (recommended)
```

09-test-vectors/rdfc/
README.md
fixtures/
wdqs-full/
case-0001/
export.rdf.nq
export.manifest.json
expected.rdf_hash.sha256
case-0002/
...
wdqs-truthy/
case-0001/
export.rdf.nq
export.manifest.json
expected.rdf_hash.sha256
ci/
rdfc-gating.json
rdfc-gating-notes.md

````

### Fixture case contents (required)
Each `case-XXXX/` MUST include:

1. `export.rdf.nq`
   - The RDF dataset export for the declared projection
   - MUST be deterministic and sorted per the RDF export profile

2. `export.manifest.json`
   - MUST declare:
     - export profile id/version (e.g., `rdf-wdqs`, projection `full` or `truthy`)
     - integrity profile enabled (`rdf-integrity-rdfc`)
     - coverage boundaries (graphs included/excluded)
     - canonicalization algorithm id/version
     - resource limits used for canonicalization
     - any skolemization policy (if applicable)

3. `expected.rdf_hash.sha256`
   - A single line containing the expected SHA-256 hash hex string (64 hex chars)
   - Example:
     - `6f2a3b...<64 hex total>...91c0`

Optional additional files:
- `notes.md` (explain special modeling cases: blank nodes, datatypes, ranks)
- `expected.canonical.nq` (canonical output, if you choose to store it)

## Naming conventions
- `wdqs-full/` and `wdqs-truthy/` correspond to the RDF WDQS Export profile projections.
- `case-0001`, `case-0002`, ... MUST be stable identifiers.
- If a fixture changes in a way that affects semantics, create a new case id rather than overwriting.

## Coverage boundaries (mandatory)
Fixtures MUST explicitly record coverage boundaries because RDFC hashing is meaningful only relative to:

- which projection is covered (`full` vs `truthy`)
- which named graphs are included (assertion/reference/etc.)
- what is excluded (metadata graphs, timestamps, etc.)

A fixture without explicit coverage declaration is non-conformant to the integrity profile.

## Resource limits (mandatory)
Because dataset canonicalization has worst-case behavior, each fixture MUST record limits used when generating the expected hash, at minimum:

- `timeout_ms`
- `max_triples` (or size cap)
- `max_blank_nodes` (or equivalent)
- `max_memory_mb` (recommended)

If a fixture exceeds limits on a reference implementation:
- it should be moved to a “stress” category and not part of default CI gating.

## CI gating configuration (recommended)
`ci/rdfc-gating.json` SHOULD define:
- which RDFC test suite (and version) is used
- which subset is gated (test IDs)
- which fixture cases are included in CI
- the resource limits used in CI

Example (shape suggestion only):
```json
{
  "rdfc_suite": { "name": "W3C RDFC-1.0", "version": "1.0" },
  "gated_tests": ["t001", "t002", "t010"],
  "fixtures": [
    { "path": "fixtures/wdqs-full/case-0001", "projection": "full" },
    { "path": "fixtures/wdqs-truthy/case-0001", "projection": "truthy" }
  ],
  "limits": {
    "timeout_ms": 30000,
    "max_triples": 500000,
    "max_blank_nodes": 200000,
    "max_memory_mb": 1024
  }
}
````

## How implementers should use this directory

1. Generate an RDF export for a fixture case using `profile-rdf-wdqs-export`.
2. Apply `profile-rdf-integrity-rdfc` to produce canonical bytes and `rdf_hash`.
3. Compare to `expected.rdf_hash.sha256`.
4. Run the CI gate subset defined in `ci/rdfc-gating.json`.

## Open questions (to finalize)

* Whether to store `expected.canonical.nq` for debugging (larger repo, but faster diagnosis).
* Whether to vendor a minimal RDFC subset locally or require implementations to pull the suite externally.
* Whether to include additional projections (beyond full/truthy) in fixtures.

```
```
