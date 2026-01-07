# Profile: RDF Integrity (RDFC) (Kristal v3)

## Status
Draft

## Purpose
Provide an optional, standardized integrity mechanism for Kristal v3 RDF exports by computing a deterministic `rdf_hash` over a specified RDF projection using an RDF Dataset Canonicalization method and CI gating.

This profile is designed for deployments that need **semantic-web grade export integrity** (independent verifiers can confirm exported RDF content matches the declared hash), while keeping Kristal v3’s normative core small.

## Scope
In scope:
- Canonicalization and hashing of RDF datasets for selected export projections
- CI conformance gating (RDFC test suite subset)
- Resource limits and fail-closed behavior for hash production

Out of scope:
- Kristal core JSON canonicalization and `kristal_id` (handled by v3 core, JCS)
- Runtime Pack hashing (this profile concerns exports, not runtime packs)
- Any requirement that this profile is enabled by default (it is optional)

## Dependencies
This profile depends on at least one deterministic RDF export profile, typically:
- `profile-rdf-wdqs-export` (full or truthy projection)

The export MUST be deterministic and byte-stable per its export profile before this integrity profile is applied.

## Normative keywords
MUST, SHOULD, MAY are used as in RFC 2119.

## Profile activation
An implementation enables this profile by declaring, in its export manifest:
- `profile.id = "rdf-integrity-rdfc"`
- `profile.version = "3.0"`
- `rdfc.algorithm = "RDFC-1.0"` (or later if explicitly listed as supported)
- `rdf_hash.alg = "sha256"`
- `rdf_hash.coverage` (explicitly enumerated projections covered)

## Covered projections (mandatory declaration)
When enabled, the implementation MUST declare exactly which RDF projection(s) are hash-covered.

At minimum, the manifest MUST include:
- `coverage.projection`: `"full"` and/or `"truthy"` (or other declared export projections)
- `coverage.export_profile_id`: e.g., `"rdf-wdqs"`
- `coverage.graphs`: which named graphs are included in the hash (assertion/reference, etc.)
- `coverage.exclusions`: e.g., “metadata graph excluded”, “generated timestamps excluded”

### Projection equivalence rules
Differences between projections (e.g., full vs truthy) MUST NOT be treated as integrity failures unless the profile explicitly requires equivalence.

Each projection’s `rdf_hash` is independently meaningful.

## Canonicalization method
The canonicalization algorithm MUST be an RDF Dataset Canonicalization method compatible with RDFC-1.0 style conformance (e.g., canonical N-Quads output).

Requirements:
- The input MUST be an RDF dataset (not a single graph unless represented as a dataset).
- The canonicalization MUST produce a deterministic canonical N-Quads byte stream (or equivalent canonical form).
- Blank nodes MUST be deterministically canonicalized (or deterministically skolemized before canonicalization, if allowed by the selected algorithm and documented).

The selected canonicalization method and version MUST be recorded in the manifest.

## Hash computation
`rdf_hash.value` MUST be computed as:

- `SHA-256(canonical_rdf_bytes)`

Where:
- `canonical_rdf_bytes` is the canonical output produced by the selected RDFC algorithm
- canonical bytes MUST be treated as a raw byte sequence (no platform-dependent normalization)

Manifest MUST include:
- algorithm identifiers
- canonicalization implementation identifier and version
- the exact export artifact(s) hashed (filenames or logical references)

## Resource limits (mandatory)
Because RDF dataset canonicalization can exhibit worst-case behavior, implementations MUST support resource limits and MUST declare them in the manifest.

Minimum required limit fields:
- `limits.timeout_ms`
- `limits.max_triples` (or dataset size cap)
- `limits.max_blank_nodes` (or equivalent)
- `limits.max_memory_mb` (recommended)

### Limit exceed behavior (fail-closed)
If resource limits are exceeded during canonicalization or hashing:
- The implementation MUST NOT emit a partial `rdf_hash`.
- The implementation MUST emit a structured failure in the export report (see Errors section).
- If the manifest declares that `rdf_hash` is present/required for the artifact, verifiers MUST treat the artifact as invalid (fail-closed).

## CI gating (mandatory when profile enabled)
If this profile is enabled in a build:
- CI MUST gate the canonicalization implementation against a selected subset of the W3C RDFC-1.0 test suite (or equivalent conformance suite declared by the implementation).
- The selected subset MUST be documented (test IDs) and versioned.

The rationale for subset selection (if not full suite) MUST be documented.

## Verification procedure
A verifier checks:
1. The export manifest declares `rdf-integrity-rdfc` profile activation, coverage, and limits.
2. The verifier re-generates (or loads) the RDF export for the declared projection.
3. The verifier canonicalizes the RDF dataset using the declared algorithm.
4. The verifier hashes the canonical bytes with SHA-256.
5. The verifier compares to `rdf_hash.value`.

If mismatch:
- Verification fails (hard failure).

If resource limits prevent verification:
- The verifier MUST report “verification not possible under declared limits”.
- If the artifact declares `rdf_hash` as required, the verifier SHOULD treat it as invalid for high-assurance contexts.

## Manifest fields (recommended shape)
Implementations SHOULD represent profile activation in the export manifest with fields similar to:

- `profiles[]`:
  - `id`
  - `version`
  - `enabled`
  - `params`

Where `params` includes:
- `rdfc.algorithm`
- `rdfc.version`
- `coverage.*`
- `limits.*`
- `rdf_hash.alg`
- `rdf_hash.value`
- `rdf_hash.artifact_ref`

## Error and warning reporting
When enabled, the exporter MUST emit structured issues:
- Errors:
  - `RDFC_CANONICALIZATION_FAILED`
  - `RDFC_LIMIT_EXCEEDED`
  - `RDFC_UNSUPPORTED_DATASET_FEATURE`
  - `RDFC_HASH_MISMATCH` (verification time)
- Warnings:
  - `RDFC_SUBSET_TEST_SUITE` (not full suite)
  - `RDFC_LIMITS_TIGHT` (limits may reduce verifiability)
  - `RDFC_SKOLIZED_BLANK_NODES` (if applicable)

Each issue SHOULD include:
- `message`
- `path` or `artifact_ref`
- `details` object

## Determinism requirements
- If the underlying export is deterministic per its export profile, and the canonicalization algorithm passes CI gating, then `rdf_hash` MUST be stable across runs given identical inputs.
- If determinism is violated (hash changes across identical rebuilds), the build MUST be treated as non-conformant under this profile.

## Conformance
A build is conformant to this profile if:
- coverage boundaries are explicitly declared,
- canonicalization and hashing follow the declared algorithms,
- resource limits are enforced and recorded,
- CI gating is applied when the profile is enabled,
- fail-closed behavior is implemented for declared `rdf_hash`.

## Interactions with other profiles
- **RDF WDQS Export** profile provides the RDF dataset this profile hashes.
- **Provenance (Nanopub + PROV-O)** profile may package similar graphs, but does not substitute for canonical dataset hashing.
- This profile MUST NOT redefine Kristal core identity (`kristal_id`); it only adds export integrity.
