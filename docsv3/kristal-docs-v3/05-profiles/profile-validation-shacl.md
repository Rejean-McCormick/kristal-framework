# Profile: Validation Reporting (SHACL)

## Status
Draft (Kristal v3 optional standardized profile)

## Purpose
Provide an interoperable **SHACL-based conformance report** for Kristal artifacts that:
- is **machine-consumable**,
- maps cleanly to the Kristal **Validation Report** issue codes/severity, and
- can be used in CI/CD as an optional validation output.

This profile is **optional** and does not change core Kristal identity or hashing.

## Profile identifier
- `profile_id`: `validation-shacl`
- `profile_version`: `1.0`

## Scope
This profile defines:
- required outputs (SHACL shapes + SHACL validation report)
- minimum mapping rules from SHACL results → Kristal validation issues
- determinism requirements for stable reporting

Non-scope:
- choosing a specific SHACL engine implementation
- mandating RDF export integrity hashing (handled by the RDFC profile)
- embedding operational patterns into Exchange/Runtime schemas

## Inputs
- A validated or candidate Kristal Exchange artifact (and/or derived RDF projection) suitable for SHACL checking
- The Kristal v3 core validation outputs (issue codes, locations) for mapping

## Outputs
When enabled, an implementation MUST produce:
1. A SHACL shapes graph (or a stable reference to it)
2. A SHACL validation report graph

Recommended serializations:
- Turtle (for shapes)
- Turtle or N-Quads / TriG (for reports)

## Normative requirements

### R1. Shapes availability (MUST)
Implementations MUST provide one of:
- `shapes_uri`: a stable URI to the SHACL shapes graph, or
- `shapes_inline`: embedded shapes graph (for offline packaging)

The shapes MUST be versioned and the version MUST be recorded.

### R2. Deterministic execution (MUST)
Given identical inputs and the same shapes version:
- the set of reported violations MUST be identical.
- ordering within serializations MUST be deterministic (stable ordering rules).

Note: SHACL engines may differ in ordering; determinism refers to the *set* of results, and deterministic serialization rules must produce byte-stable outputs when the engine output set is the same.

### R3. Required linkage to Kristal validation report (MUST)
The SHACL report MUST be linkable to the Kristal Validation Report via:
- `kristal:kristalId` (or equivalent) and/or `kristal:buildId`
- and MUST declare `profile_id=validation-shacl` in the Validation Report `profiles` list (or equivalent).

### R4. Mapping rules (MUST)
Each SHACL validation result MUST map to a Kristal Validation Report issue:

- `sh:Violation` → `severity = ERROR`
- `sh:Warning` → `severity = WARNING`
- `sh:Info` → `severity = INFO`

And must map to:
- `code`: a stable Kristal issue code
- `message`: a human-readable explanation
- `location`: a JSON pointer (if mapping to JSON) and/or an RDF focus node IRI

### R5. Minimum SHACL result fields (MUST)
Each SHACL result node SHOULD include, and implementations MUST be able to produce, at minimum:
- `sh:focusNode`
- `sh:resultSeverity`
- `sh:sourceShape` or `sh:sourceConstraintComponent`
- `sh:resultMessage`

### R6. No effect on compilation gating unless configured (MUST)
Enabling SHACL output MUST NOT change core gating behavior by itself.
Core gating remains driven by the deterministic Kristal validator.
If a deployment chooses to use SHACL as an additional gate, this MUST be explicitly configured and documented (non-default).

## Recommended conventions (SHOULD)

### C1. Shape naming and versioning (SHOULD)
- Shapes SHOULD use stable IRIs containing:
  - `spec_version`
  - `shape_version`
- Example: `urn:kristal:shapes:v3:exchange:1.0`

### C2. Code system (SHOULD)
Define a code namespace for SHACL-mapped issues:
- `KRS_V3_<CATEGORY>_<NAME>`
Examples:
- `KRS_V3_SCHEMA_MISSING_REQUIRED_FIELD`
- `KRS_V3_EVIDENCE_MISSING`
- `KRS_V3_VALUE_NORMALIZATION_FAILED`

### C3. Include both JSON and RDF pointers where possible (SHOULD)
If the validator can derive a JSON pointer for the corresponding location in Exchange, include it in:
- Kristal Validation Report `location.json_pointer`
- SHACL result message or an extension triple (see below)

## Report structure requirements

### Shapes graph
- MUST contain SHACL shapes (NodeShapes / PropertyShapes)
- MUST declare an explicit version identifier (via `owl:versionInfo` or a Kristal predicate)

### Report graph
- MUST follow SHACL Validation Report structure:
  - `sh:conforms` boolean
  - `sh:result` list of result nodes

## Required extension predicates (recommended)
To improve interop between SHACL outputs and Kristal issue locations, implementations SHOULD emit:
- `kristal:issueCode` on each `sh:ValidationResult`
- `kristal:jsonPointer` on each `sh:ValidationResult` when possible
- `kristal:claimId` / `kristal:evidenceId` when applicable

These extensions must not be required by generic SHACL tools but make downstream automation reliable.

## Failure modes and required behaviors
- If shapes are unavailable or invalid, the system MUST fail the SHACL profile execution and record an ERROR in the Kristal Validation Report (but core validation may still proceed if configured).
- If SHACL execution exceeds configured limits (timeout/memory), the profile MUST fail and record an ERROR with explicit limit information.

## Conformance tests
A conforming implementation MUST provide fixtures demonstrating:
- stable shapes versioning and reproducible report production
- correct severity mapping (Violation/Warning/Info)
- stable mapping to Kristal issue codes
- linkability (kristal_id/build_id present)
- deterministic serialization rules for the chosen format(s)

## Open questions (to finalize)
- Do we standardize a single shapes vocabulary per artifact type (Exchange vs Runtime Pack), or allow multiple named shape sets?
- Do we require engines to output `sh:sourceConstraintComponent` consistently, or treat it as best-effort?
