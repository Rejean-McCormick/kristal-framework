# Glossary

A compact reference of Kristal terms used across the wiki. When a term has a formal definition in the specs, the wiki keeps only the operational meaning.

---

## Artifact
A packaged, distributable unit produced by the Kristal pipeline (e.g., Exchange, Runtime Pack). Artifacts are designed to be verifiable and safe to mirror.

## Authority
An entity allowed to seal/attest an artifact for a given scope. Authorities are evaluated against trust roots and scope rules (often via an Authority Registry).

## Authority Registry
A pinned, versioned policy artifact that defines trust roots and scope-based acceptance rules for authorities. Used to verify shards and federation roots offline.

## Build
A deterministic compilation run producing artifacts. A build records enough inputs, policies, tooling identity, and canonicalization choices to make outputs reproducible.

## Canonicalization
A standard way to turn structured data into a consistent byte representation so hashes and signatures are stable across implementations.

## Capability
A declared feature of a query runtime (operators, pagination mode, limits, estimates). Clients use capabilities to adapt or fail early.

## Claim-IR
An intermediate representation of extracted claims before deterministic resolution. Used as a stable input stage in the pipeline.

## Composition Policy
Deterministic rules for composing multiple shards into one federated view (precedence, conflict handling, ordering).

## Consumer
Any system or client that retrieves and verifies Kristal artifacts and uses them (query engines, offline tools, serving systems).

## Determinism
The property that the same inputs + policies + tooling configuration produce the same canonical outputs and IDs.

## Distribution
The mechanism used to publish and retrieve artifacts (stores, mirrors, indexes). Distribution should support offline retrieval and verification.

## Exchange
The portable bundle of curated knowledge produced by compilation. It is the core artifact from which runtime packs are built.

## Fail-closed
A verification posture where declared integrity requirements (hashes/signatures/evidence) must verify, otherwise the artifact is rejected.

## Federation
A composed Kristal view built from multiple shards (often multiple publishers). Federation must be deterministic and verifiable.

## Federation Manifest
The manifest describing a federation root: which shards it includes and how they are composed (composition policy + publisher seal).

## Identity (ID)
A content-derived identifier for an artifact. If meaningful content changes, the ID changes. If content is the same, the ID should be the same.

## Integrity
The set of mechanisms used to detect tampering or mismatch (hashes, signatures, evidence references).

## Manifest
A metadata document describing an artifact: what it contains, what inputs/policies produced it, and how to verify it.

## Offline-correct
The ability to verify and use artifacts without contacting remote services, relying on declared integrity evidence and pinned policy.

## Policy
Recorded parameters that affect outputs (ordering, grouping, membership filters, bitmap strategy). Policies must be explicit for reproducibility.

## Profile
A named set of requirements or behaviors layered on top of core (e.g., projection format, validation suite). Profiles enable compatibility and evolution.

## Publisher
The party that publishes an artifact to distribution (may also be the authority, or may be distinct in curated federation scenarios).

## Query Contract
The agreed interface semantics for querying a runtime pack (capabilities, pagination, operators, constraints).

## Resolved Claim-IR
A deterministically resolved form of Claim-IR that removes ambiguity and makes outputs reproducible.

## Rollback
Switching activation back to a previously known-good artifact (typically by changing an activation pointer to a previous Runtime Pack ID).

## Runtime Pack
A query-optimized package built from an Exchange (indexes, bitmaps, policies) intended for local or service-based querying.

## Seal
A signature-based attestation over an artifact or manifest (often used for shard integrity and authority claims).

## Shard
A scoped Kristal artifact (e.g., by domain/authority/time window) intended to be composed with other shards.

## Shard Manifest
The manifest describing a shard: its scope, inputs, validation evidence, integrity, and authority seal(s).

## Trust Root
A pinned anchor for verification (key/cert/DID or equivalent). Trust roots are used to decide which authorities can be accepted.

## Validation
Checks applied to artifacts (core rules + profile checks). Validation is typically recorded as Validation Reports.

## Validation Report
An artifact recording validation outcomes, references, and integrity information used for auditability and fail-closed verification.