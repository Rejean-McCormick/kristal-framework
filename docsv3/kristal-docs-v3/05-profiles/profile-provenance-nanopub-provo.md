# Profile: Provenance Packaging (Nanopublication + PROV-O)

## Status
Draft (Kristal v3 optional standardized profile)

## Purpose
Provide a portable, interoperable way to package **assertions + provenance + publication metadata** for Kristal exports using:
- **Nanopublication structure** (named graphs: Head, Assertion, Provenance, PubInfo)
- **PROV-O** relations for provenance modeling

This profile is **optional** and MUST NOT affect core Kristal identity/hashing unless explicitly declared by a separate integrity profile.

## Scope
This profile defines:
- The required named graphs and minimum required triples for a nanopublication
- How Kristal claim/evidence identifiers map into nanopub/provenance structures
- How to reference source artifacts (Exchange, Runtime Pack) and build metadata
- Interop rules (stable IRIs, deterministic serialization requirements)

Non-scope:
- Defining trust roots, key distribution, or signature algorithms (handled by signing/trust docs)
- Mandating RDF dataset hashing (handled by the RDFC export integrity profile)
- Mandating full PROV completeness beyond the minimal required terms

## Profile identifier
- `profile_id`: `provenance-nanopub-provo`
- `profile_version`: `1.0`

## Inputs
- A **validated** Kristal Exchange artifact (and optionally a Runtime Pack)
- Deterministic export projection (RDF export profile) to which nanopubs refer
- Claim-level identifiers (`claim_id`) and evidence identifiers (`evidence_id`) as produced by the Kristal pipeline

## Outputs
- One or more nanopublications, each expressed as RDF with named graphs:
  - `:Head`
  - `:Assertion`
  - `:Provenance`
  - `:PubInfo`

Recommended serialization:
- N-Quads (preferred for transport and tooling)
- TriG (acceptable)

## Normative requirements

### R1. Graph structure (MUST)
Each nanopublication MUST contain exactly these four named graphs:

1. **Head graph**: links the nanopublication to the other graphs
2. **Assertion graph**: contains the assertion triples
3. **Provenance graph**: contains provenance about the assertion
4. **PubInfo graph**: contains publication metadata (creator, time, software, etc.)

### R2. Stable identifiers (MUST)
Nanopublication and graph IRIs MUST be stable and deterministic for a given claim bundle.

A conforming implementation MUST use one of the following strategies:

**Strategy A (recommended): content-addressed nanopub IRI**
- `np_id = sha256(canonical_representation_of(assertion_bundle))`
- Base IRI: `urn:kristal:np:<np_id>`
- Graph IRIs:  
  - `urn:kristal:np:<np_id>#Head`  
  - `urn:kristal:np:<np_id>#Assertion`  
  - `urn:kristal:np:<np_id>#Provenance`  
  - `urn:kristal:np:<np_id>#PubInfo`

**Strategy B: deterministic hierarchical IRIs**
- Base IRI derived from `kristal_id` + `claim_id` (or statement group id)
- Graph IRIs formed by suffixing `#Head|#Assertion|#Provenance|#PubInfo`

The chosen strategy MUST be declared in the export metadata.

### R3. Assertion mapping (MUST)
The Assertion graph MUST correspond to one of:
- a single Kristal `claim_id`, or
- a deterministic bundle of claims (e.g., all claims for an entity), provided bundling rules are deterministic and recorded.

The Assertion graph MUST contain:
- the RDF triples representing the claim(s) under the selected RDF export profile
- links back to Kristal claim identifiers via `kristal:claimId` (or an equivalent stable predicate)

### R4. Provenance mapping (MUST)
The Provenance graph MUST express:
- at least one provenance link from the assertion to its evidence
- evidence must be identified using deterministic IDs (`evidence_id`) and/or stable source URIs

The minimal requirement is:
- the assertion (or a proxy node for the assertion) `prov:wasDerivedFrom` the evidence node(s)

### R5. Publication info (MUST)
The PubInfo graph MUST include:
- `prov:generatedAtTime` (UTC timestamp)
- `prov:wasAttributedTo` (agent / organization)
- a pointer to the software/tooling version that produced the nanopub

### R6. Deterministic emission (MUST)
Given identical inputs and policy selections, implementations MUST emit byte-stable nanopub datasets under:
- the same serialization format, and
- deterministic ordering rules for triples/quads within each named graph.

Note: this requirement is about deterministic export generation. Hashing the RDF dataset is not required by this profile.

### R7. No changes to core identity (MUST)
Enabling this provenance packaging profile MUST NOT change:
- `kristal_id`
- Exchange `content_hash`
- Runtime Pack content hashes

unless a separate integrity profile explicitly includes these exports in hash coverage.

## Recommended conventions (SHOULD)

### C1. One nanopub per claim (SHOULD)
Default behavior SHOULD be **one nanopublication per `claim_id`** for maximal addressability and incremental updates.

### C2. Include pointers to Exchange and build metadata (SHOULD)
PubInfo SHOULD include:
- `kristal:kristalId` (the Exchange id)
- `kristal:buildId`
- compiler identity/version fields

### C3. Use PROV-O consistently (SHOULD)
Use PROV-O terms:
- `prov:Entity` for evidence artifacts
- `prov:Activity` for build steps (extract, resolve, validate, compile)
- `prov:Agent` for organizations/services

## Minimal required triples (templates)

### Head graph (required pattern)
- `np: <npIRI> np:hasAssertion <assertionGraphIRI> .`
- `np: <npIRI> np:hasProvenance <provenanceGraphIRI> .`
- `np: <npIRI> np:hasPublicationInfo <pubInfoGraphIRI> .`

(Exact nanopub predicates depend on chosen nanopub vocabulary; implementation MUST declare vocabulary IRIs in export metadata.)

### Provenance graph (required pattern)
- `<assertionNode> prov:wasDerivedFrom <evidenceNode> .`

### PubInfo graph (required pattern)
- `<npIRI> prov:generatedAtTime "YYYY-MM-DDThh:mm:ssZ"^^xsd:dateTime .`
- `<npIRI> prov:wasAttributedTo <agentIRI> .`
- `<npIRI> kristal:compilerVersion "..." .`

## Interoperability constraints
- Implementations MUST document:
  - which nanopub vocabulary is used (IRIs),
  - the IRI strategy (A or B),
  - bundling rules if bundling is used,
  - the serialization format and ordering rules.

## Failure modes and required behaviors
- If the underlying Exchange fails validation, nanopub emission MUST NOT proceed.
- If an evidence link is missing for a claim and the validator marks it as ERROR, nanopub emission MUST NOT proceed for that claim.
- If an evidence link is missing but marked WARNING, nanopub emission MAY proceed, but MUST include a warning issue in the Validation Report.

## Conformance tests
A conforming implementation MUST provide fixtures demonstrating:
- deterministic nanopub IRI generation (same inputs → same IRIs)
- deterministic dataset emission (same inputs → byte-stable quads)
- correct mapping of claim_id and evidence links into PROV-O relations
- enforcement that this profile does not alter core identity/hashing

## Open questions (to finalize)
- Which nanopub vocabulary will be standardized (or should we support a small allowed set)?
- Should we require one nanopub per claim as a MUST, or keep it as SHOULD?
- Do we allow multiple evidence nodes per claim in provenance, and how do we order them deterministically?
