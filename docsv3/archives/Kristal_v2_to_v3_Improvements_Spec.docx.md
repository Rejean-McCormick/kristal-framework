**Kristal v3**

## *Improvements from v2 to v3*

Date: January 07, 2026  
Status: Draft (implementation-facing)  
Audience: Orgo / SenTient / Architect / Konnaxion implementers and reviewers

# **1\. Executive summary**

Kristal v3 tightens Kristal v2 into a more interoperable, reproducible, and integration-ready standard. v3 introduces a small mandatory core for canonicalization, identity, integrity, manifests, and inter-system contracts; everything else is expressed as explicit profiles and non-normative operational guidance to preserve adoption velocity while enabling high assurance deployments.

This document specifies what changes between v2 and v3, what is mandatory vs optional, how conformance is tested, and how v3 integrates first into the Konnaxion × Orgo × Architect × SenTient ecosystem.

# **2\. Principles and scope**

## **2.1 Design principles**

* Small normative core, strong defaults: keep mandatory requirements minimal but unambiguous.  
* Profiles for optional capabilities: performance, provenance, and semantic integrity extensions are standardized but optional.  
* Interoperability first: two independent implementations must produce identical IDs for identical artifacts.  
* Fail-closed integrity: if an artifact declares hashes or signatures, verification failure is a hard error.  
* Offline-first execution: Runtime Packs remain offline-executable with constrained query semantics (no full SPARQL).  
* Deterministic late: probabilistic systems may propose (Claim-IR), but acceptance/compilation/rendering must be deterministic.

## **2.2 Scope boundaries**

* Kristal Exchange remains the canonical, mergeable, content-addressed source of truth.  
* Kristal Runtime Pack remains derived, indexed, offline-executable, and intentionally does not implement full SPARQL semantics.  
* Operational architecture patterns (circuit breakers, DLQs, blue-green, etc.) are documented as non-normative guidance and are not embedded as schema-level objects.

# **3\. Summary of changes (v2 → v3)**

| Area | v2 | v3 improvement |
| :---- | :---- | :---- |
| Canonicalization | Custom / underspecified | RFC 8785 JCS adopted as normative canonical\_json; profile/version recorded |
| Identity | Hashing present, cross-tool drift risk | Content hash \= SHA-256(JCS-canonical JSON); optional Trusty/ni representation |
| Integrity semantics | Mixed failure handling | Fail-closed when hashes/signatures are declared but do not verify |
| Reproducibility | Best-effort packs | Reproducible pack builds as acceptance criteria; minimal reproducibility manifest required |
| Exports | Basic JSON/RDF intent | Deterministic JSON-LD 1.1 profile and WDQS-compatible RDF export profile; optional RDF integrity profile (RDFC) |
| Runtime performance knobs | Partially described | Enumerated portable policies (ordering/row-groups/filters/bitmaps) recorded in manifest; advanced knobs moved to profiles |
| Query surface | Constrained queries | Optional TPF-like pagination \+ per-pattern cardinality; standardized join caps portability profile |
| Provenance packaging | Evidence present, packaging ad hoc | Optional nanopublication \+ PROV-O packaging profile |
| Compliance | Examples, limited gating | Golden test vectors and CI gating; optional RDFC test suite gating with resource limits |

# **4\. v3 normative core (mandatory)**

The following requirements are mandatory for v3 conformance. Everything not listed here must be treated as optional via standardized profiles or non-normative guidance.

## **4.1 Canonicalization and content-addressed identity**

* canonical\_json MUST be defined as RFC 8785 JSON Canonicalization Scheme (JCS).  
* kristal\_id MUST be computed as SHA-256 over canonical\_json of the Exchange artifact with signatures excluded.  
* Artifacts MUST record canonicalization\_profile and canonicalization\_version (or equivalent) in the manifest so implementations can compare IDs across toolchains.  
* Implementations MUST publish and pass golden test vectors for canonicalization and hashing.

## **4.2 Integrity, signatures, and fail-closed behavior**

* If an artifact declares a hash, signature, or signer identity, verifiers MUST fail closed when verification fails.  
* Signing workflow MUST be unambiguous: remove signatures → canonicalize → hash → verify/sign.  
* Runtime Pack distributors/consumers MUST verify signatures before use when signatures are present.

## **4.3 Minimal reproducibility manifest**

v3 requires a minimal manifest sufficient to reproduce Exchange and Runtime Pack outputs from the same inputs and compiler parameters.

* Manifest MUST include: input snapshot identifiers, compiler identity and version, config hash, build\_id, and timestamps.  
* Manifest MUST include the portable policy selections for ordering, row-grouping, membership filters, and bitmap optimization (see Section 6).  
* Unknown manifest fields MUST be ignored by default for forward compatibility, except integrity fields (hash/signature declarations) which remain fail-closed.

## **4.4 Inter-system interface contracts**

v3 formalizes contracts between the systems in the ecosystem. These contracts are mandatory and enforced by schema validation and deterministic rules.

* Claim-IR schema contract: the only allowed direct output format for extractors; includes uncertainty and evidence pointers.  
* Resolution output contract (SenTient): ranked QID/PID candidates, normalized literals, and explicit unresolved states (never silently coerced).  
* Validation contract: deterministic rules with structured error/warning codes; failed validation MUST block compilation.  
* Exchange commit contract: defines exactly what is hashed/signed and how merges/incremental updates are represented.  
* Runtime Pack manifest contract: defines what affects offline behavior and is required for reproducibility.

## **4.5 Deterministic export profiles (baseline)**

* Implementations MUST provide deterministic JSON-LD 1.1 export under a stable @context/profile.  
* Implementations MUST provide a WDQS-compatible RDF export profile, including a defined truthy/best-rank mapping behavior.  
* Baseline exports MUST be deterministic (byte-stable given identical inputs and policy selections).

# **5\. Standardized optional profiles**

v3 standardizes optional capabilities as explicit profiles. Profiles are interoperable add-ons: they must be precisely specified when enabled, but are not required for core conformance.

| Profile | Purpose | Key requirements when enabled | Notes |
| :---- | :---- | :---- | :---- |
| Export integrity (RDFC) | RDF-level integrity hashing for exports | Compute rdf\_hash via RDFC-1.0 canonical N-Quads; gate with W3C tests; enforce resource limits | Optional; expensive worst-case |
| Provenance packaging (Nanopub \+ PROV-O) | Portable provenance and attribution packaging | Emit nanopublication structure (head \+ assertion/provenance/pubinfo graphs) using PROV-O relations | Optional; can increase size |
| Validation reporting | Machine-readable conformance reports | Emit optional SHACL ValidationReport and/or constrained ShEx report mapping to ERROR/WARNING codes | Optional; choose primary output |
| Runtime performance extensions | Higher performance or observability knobs | Optional Parquet Bloom filters; optional bitmap container stats; optional domain partitioning metadata | Optional; keep core portable |
| Query surface extensions | More ergonomic integration tooling | TPF-like pagination and per-pattern cardinality (exact or approximate); defined cursor semantics | Optional; keeps Runtime offline |

## **5.1 Profile boundaries for RDF integrity (RDFC)**

When enabled, the RDF integrity profile MUST explicitly declare which RDF projection(s) are covered by rdf\_hash (e.g., full statement graph vs truthy projection), and what constitutes equivalence. Differences between projections MUST NOT be treated as integrity failures unless the profile explicitly requires equivalence.

## **5.2 Profiles vs core: keeping the determinism surface small**

To preserve adoption and credibility, v3 treats most tunability as profiles or non-normative guidance. Core conformance requires recording a small set of portable policies; profiles can add extra knobs, but must not redefine core identity or validation rules.

# **6\. Portable policy set (v3)**

Rather than allowing arbitrary performance strategies, v3 defines a small enumerated set of portable policies. Implementations MUST select from these policies and record the selection(s) in the manifest. This keeps packs reproducible and practically comparable across implementations.

## **6.1 Triple table ordering policies**

Allowed orderings for the Parquet triples table (treat ordering as an index):

* SPO, SOP, PSO, POS, OSP, OPS (classic RDF triple-table orderings).  
* Composite order sets are allowed (e.g., {SPO, POS}) and MUST be declared explicitly.  
* Stable tie-breakers MUST be defined (e.g., s then p then o, with claim\_id as last key) to make ordering deterministic.

## **6.2 Row-group sizing policies**

* FIXED\_ROWS(n): constant row count per row group.  
* FIXED\_BYTES(n): constant target byte size per row group.  
* ADAPTIVE(target\_rows, target\_bytes): bounded adaptive policy with deterministic rules.  
* Policy parameters MUST be recorded; implementations SHOULD provide recommended defaults.

## **6.3 Membership filter policies**

* Default filter type SHOULD be binary-fuse for immutable sets.  
* Allowed filter families: binary-fuse (recommended), xor (optional).  
* Portable parameters MUST include: target\_false\_positive\_rate, variant (3-wise or 4-wise), seeds, bits-per-key (or equivalent), and what key space is filtered (claim\_id vs spo\_hash, etc.).  
* False positives MUST be pruned deterministically using the underlying authoritative data (no probabilistic acceptance).

## **6.4 Bitmap index policies**

* Allowed bitmap format: Roaring bitmaps.  
* Run optimization (Roaring+Run) SHOULD be applied where beneficial; if applied, it MUST be recorded.  
* Optional container statistics may be recorded as a profile extension (not required by core).

## **6.5 Join caps and strictness**

* Implementations MUST declare a join1\_cap policy (default cap and strict mode behavior) to reduce cross-implementation divergence.  
* Strict mode MUST be deterministic and MUST specify failure behavior (error vs truncated results) in the query contract.

# **7\. Offline query contract**

Runtime Packs remain offline-executable and intentionally constrained. v3 standardizes the minimum required query behavior and keeps richer interfaces as optional profiles.

## **7.1 Core query requirements (mandatory)**

* Triple-pattern query over (s, p, o) with any of the three fields bound/unbound.  
* Deterministic result ordering for a given ordering policy selection.  
* Stable cursor semantics for paging OR stable offset semantics (implementation MUST pick one and document it).  
* Well-defined behavior for ranks/truthy semantics (baseline: provide truthy mapping in export; runtime may optionally expose truthy projection as a profile).

## **7.2 TPF-like pagination and cardinality (optional profile)**

When the query surface extension profile is enabled, the Runtime Pack or local HTTP wrapper SHOULD provide TPF-like fragments with cursor-based pagination and per-pattern cardinality metadata (exact or approximate) to support predictable tooling integration.

# **8\. Conformance and compliance gating**

## **8.1 Golden test vectors (mandatory)**

* JCS canonicalization vectors (including edge cases) and expected SHA-256 hashes.  
* Exchange and Runtime Pack example fixtures with expected kristal\_id values.  
* Query fixtures that validate paging and join-cap behavior.

## **8.2 RDFC gating and resource limits (optional profile)**

* If the RDFC profile is enabled, CI MUST gate against the selected subset of the W3C RDFC-1.0 test suite.  
* Implementations MUST enforce resource limits (timeouts/iteration caps) and document coverage and limits.  
* If resource limits are exceeded, behavior MUST be fail-closed for rdf\_hash production (no partial hashes).

# **9\. First integration in the ecosystem**

v3 is first integrated as the portable, offline-executable knowledge substrate within Konnaxion × Orgo × Architect × SenTient. The contracts in Section 4.4 define the boundaries.

## **9.1 Orgo (workflow and governance control plane)**

* Orgo owns the build workflow state: ingest → extract (Claim-IR) → resolve (SenTient) → validate → publish.  
* Orgo enforces “no compile on validation failure” and records build metadata (build\_id, inputs, signatures, distribution status).  
* Orgo keeps audit logs, approvals, and distribution state; Kristal artifacts remain the knowledge payload.

## **9.2 SenTient (resolver)**

* SenTient emits Resolved Claim-IR conforming to the resolution output contract.  
* SenTient MUST preserve unresolved ambiguity explicitly and MUST NOT silently coerce values.  
* Operational guidance (non-normative): use circuit breakers and timeouts; on failure, return unresolved states and warnings rather than blocking pipelines.

## **9.3 Architect (deterministic renderer)**

* Architect consumes only validated Kristal query outputs.  
* Architect MUST not introduce new facts and MUST trace every asserted sentence to claim\_id and evidence pointers.  
* Architect output SHOULD include a machine-readable trace map for audits.

## **9.4 Konnaxion (distribution and offline UX)**

* Konnaxion distributes signed Runtime Packs as versioned offline packages (PWA cache / low-bandwidth modes).  
* Konnaxion clients MUST verify signatures when present, enforce pinned trust roots per tenant, and apply rollback/downgrade prevention policies.  
* Collective-intelligence workflows (Ekoh/Konsensus/Smart Vote) SHOULD influence promotion and distribution priority, not mutate canonical facts.

# **10\. Multi-tenancy and feedback loop**

## **10.1 ID and access model**

* kristal\_id remains globally content-addressed (same content → same ID).  
* Access control and distribution are layered above IDs (tenant-scoped keys and channels).  
* Signing keys MAY be tenant-scoped even when IDs are global.

## **10.2 Mutation path (mandatory)**

* Feedback MUST NOT edit Exchange directly.  
* All changes MUST flow: signal → Claim-IR → resolution → validation → new Exchange commit → new Runtime Pack build.  
* Curation/votes MUST operate on promotion/distribution decisions, not facts.

# **11\. Non-normative implementation notes and operational guidance**

This section uses “senior architecture patterns” style meta structures as documentation and system-design checklists (concept → problem → solution → when to use → pitfalls → observability). These patterns are NOT schema-level constructs and do not affect conformance.

## **11.1 Failure-path patterns (recommended)**

* Circuit breaker \+ timeout around SenTient resolution calls; preserve unresolved ambiguity on failure.  
* Dead-letter queue (DLQ) for ingestion/build queues; quarantine poison inputs instead of infinite retry.  
* CQRS alignment: Exchange as write/command model; Runtime Pack as read model.  
* Blue-green or canary release for Runtime Pack distribution to reduce blast radius of bad releases.  
* Structured logs \+ correlation IDs across stages (build\_id, kristal\_id, claim\_id) to accelerate debugging.

## **11.2 Observability checklist (recommended)**

* Every pipeline stage emits structured logs including build\_id, stage, input snapshot IDs, and outcome.  
* Validation errors use stable machine-readable codes with human-readable messages.  
* Pack publish events include signature verification status and version metadata.  
* Resource limit breaches (e.g., RDFC) are surfaced as explicit events and metrics.

# **12\. Migration plan (v2 → v3)**

## **12.1 Compatibility**

* v3 is additive where possible: new fields (canonicalization\_profile, rdf\_hash, policy selections, validation reports) do not invalidate v2 readers that ignore unknown fields.  
* v3 requires JCS: v2 canonicalization remains acceptable only if it is JCS-compatible.  
* Readers MUST ignore unknown fields but MUST fail closed on declared integrity fields that do not verify.

## **12.2 Deliverables and definition of done**

1. Updated v3 schemas: Claim-IR, Resolved Claim-IR, Validation Report, Exchange Manifest, Runtime Pack Manifest.  
2. Reference canonicalizer and hash/sign tooling with golden vectors.  
3. Compiler updates producing reproducible packs with recorded portable policies.  
4. Deterministic JSON-LD and WDQS-compatible RDF export profiles.  
5. Optional profiles implemented behind feature flags (RDFC, Nanopub+PROV-O, SHACL/ShEx, query extensions).  
6. Conformance checklist and migration guide for v2 implementers.

# **Appendix A. Conformance checklist (v3 core)**

* JCS canonicalization implemented; golden vectors pass.  
* kristal\_id computed as SHA-256(JCS-canonical JSON) excluding signatures.  
* Fail-closed verification implemented for declared hashes/signatures.  
* Minimal reproducibility manifest emitted with required build metadata and policy selections.  
* Claim-IR, resolution output, and validation contracts enforced by schema \+ deterministic rules.  
* Exchange and Runtime Pack outputs deterministic under selected policies.

# **Appendix B. Optional profile checklist**

* RDFC export integrity: rdf\_hash computed; W3C RDFC tests gated; resource limits enforced.  
* Nanopub \+ PROV-O packaging: three graphs emitted with required links and PROV-O relations.  
* SHACL/ShEx reporting: validation reports emitted and mapped to Kristal error codes.  
* Query surface extensions: TPF-like paging \+ cardinality metadata implemented with stable cursor semantics.