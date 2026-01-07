# Kristal v3 Improvements Plan

*Updated to include literature-driven improvements (add6).*

Date: 2026-01-07

# **1\. Executive summary**

v3 upgrades the v2 draft spec into a tighter, more interoperable, and reproducible standard. This revision incorporates additional performance and provenance improvements derived from recent literature (Parquet-based RDF storage, Roaring+Run, binary-fuse filters, nanopublications/PROV-O, RDFC test gating, and high-throughput JSON ingestion patterns).

## **Key outcomes**

* Normative canonicalization and hashing that is portable across languages (JSON first, optional RDF-level hashing).  
* Stronger RDF/JSON-LD interoperability: stable JSON-LD 1.1 context/profile and WDQS-compatible RDF export.  
* Runtime pack reproducibility: fully specified membership filter metadata, bitmap conventions, and Parquet ordering/row-group policy.  
* More ergonomic offline query surface: Triple Pattern Fragment (TPF)-like pagination \+ per-pattern cardinality metadata.  
* First-class “subset recipes” and compiler build metadata so offline subsets become reproducible artifacts.  
* Portable provenance and trust packaging via an optional nanopublication/PROV-O shaped mode for Exchange exports.

# **2\. Workstreams and planned improvements**

## **2.1 Canonicalization, identity, integrity, and compliance gating**

* Adopt RFC 8785 (JCS) as the normative definition of canonical\_json to remove ambiguity in primitive serialization and edge cases.  
* Standardize signing/verification workflow: remove signatures → canonicalize → hash/verify; define failure modes (fail-closed).  
* Add optional RDF integrity for exports using RDF Dataset Canonicalization (RDFC-1.0) and an rdf\_hash computed from canonical N-Quads.  
* Record canonicalization profile/version in Exchange and pack manifests (so hashes remain comparable across toolchains).  
* Gate RDF canonicalization compliance in CI using the RDFC-1.0 W3C test suite (eval/map tests) and publish Kristal test vectors.  
* Introduce resource limits for RDF canonicalization to mitigate worst-case datasets (timeouts / iteration caps).  
* Add optional Trusty/ni-URI-style identifier form as a human-usable, verifiable representation of kristal\_id (or rdf\_hash when in RDF mode).

* Acceptance criteria:  
* Two independent implementations produce identical kristal\_id for the same Exchange artifact.  
* If rdf\_hash is present, semantically identical RDF projections produce identical hashes.  
* Reference implementation passes the selected RDFC-1.0 test suite subset with documented coverage.

## **2.2 RDF / JSON-LD interoperability and provenance packaging**

* Publish a canonical JSON-LD 1.1 profile: stable @context, framing expectations, and dataset/graph mapping for provenance/evidence.  
* Define a WDQS-compatible RDF export profile (statement-node IRIs, prefix mapping, and which ranks map to wdt: truthy triples).  
* Add an optional “nanopub mode” export for Kristal Exchange: head graph linking (assertion graph \+ provenance graph \+ publication-info graph).  
* Use PROV-O terms in nanopub mode (prov:Entity/Activity/Agent, prov:used, prov:wasGeneratedBy, prov:wasDerivedFrom, prov:wasAttributedTo), so claims are portable and attributable.  
* Extend/confirm entity-id encoding for Lexeme/Form/Sense and non-item entity types; ensure round-trip safety.

* Acceptance criteria:  
* Exchange → RDF/JSON-LD export is deterministic and documented; JSON-LD and RDF exports round-trip through common tooling.  
* Nanopub mode produces three graphs with required links and PROV-O predicates, and is hashable via rdf\_hash.

## **2.3 Validation and quality reporting**

* Add an optional SHACL output profile that emits sh:ValidationReport for core constraints (Exchange structural \+ semantic checks).  
* Define a constrained Kristal ShEx profile (subset) and specify which graph is validated (RDF export vs internal statement graph) and how violations map to ERROR/WARNING codes.  
* Clarify compile-time rules around validation\_status: failed must not compile; partial may compile with explicit unresolved handling and explicit warnings.

* Acceptance criteria:  
* A pack compiler can emit machine-readable validation reports (Kristal codes \+ optional SHACL report graph).

## **2.4 Runtime pack reproducibility and performance (Parquet \+ indexes \+ filters)**

* Parquet triples table: treat ordering as an index. Record the sort order(s) used (e.g., SPO/PSO/POS, etc.) and the row-group sizing policy in manifest.json because these control BRI (min/max “zone map”) effectiveness.  
* Enable optional Parquet-level Bloom filters on dictionary-encoded IDs (e.g., p and/or (p,o)) to prune row groups during scans, complementing external membership filters.  
* Membership filters: make binary-fuse the recommended default for immutable sets; standardize default false-positive-rate targets per use case and record construction parameters (variant 3-wise/4-wise, seeds, bits/key).  
* Clarify what membership filters gate (claim\_id vs spo\_hash vs other identifiers) and how false positives are pruned deterministically.  
* Roaring bitmaps: require run optimization (Roaring+Run) where beneficial; optionally record container statistics per index to inform heuristics.  
* Data clustering: allow (and document) row reordering / ID assignment strategies that increase runs, improving both Roaring+Run compression and Parquet BRIs.

* Acceptance criteria:  
* Given the same Exchange input and compiler parameters, pack binaries are reproducible (or a reproducibility mode is defined).  
* Manifest captures Parquet ordering \+ row-group policy \+ Bloom/filter settings sufficient to reproduce scan pruning behavior.  
* Binary-fuse construction parameters are fully specified and validation checks FPR bounds.

## **2.5 Query interface evolution**

* Add TPF-like affordances: pagination for triple-pattern results and per-pattern cardinality estimates (exact or approximate) to support predictable tooling integration.  
* Add an optional compiled “truthy/best-rank” projection so default offline behavior matches WDQS wdt: semantics (with an escape hatch to query all ranks).  
* Define a portability profile for join1 caps (default cap \+ strict mode) to reduce cross-implementation divergence.

* Acceptance criteria:  
* A minimal HTTP (or local) fragment API can expose triple-pattern pages with stable cursor semantics; offline file-based execution remains supported.

## **2.6 Subset builder, compiler pipeline, and ingestion throughput**

* Make “subset recipes” first-class: seeds (entities/properties), deterministic expansion rules, allow/deny lists, depth limits, stopping criteria, and snapshot identifiers.  
* Formalize a pack compiler pipeline: official dumps/exports → Exchange (canonical) → Runtime (triple table \+ indexes \+ filters).  
* Add ingestion guidance: require compilers to support field-skipping / on-demand parsing of Exchange JSON/JSON-LD; recommend two-stage parsing strategies (structural scan then materialization) for throughput.  
* Record build throughput metrics in build metadata (inputs size, wall time, peak memory), so compilation cost becomes predictable and comparable.  
* Optional domain partitioning for large packs (coarse splits) as a supported distribution strategy.

* Acceptance criteria:  
* Two builds with the same recipe \+ source snapshot produce identical subset outputs (modulo declared non-determinism).  
* Compilers support skipping irrelevant fields and demonstrate streaming build capability in the reference implementation.

# **3\. Roadmap and deliverables**

| Phase | Deliverables | Definition of done | Notes / risks |
| :---- | :---- | :---- | :---- |
| P0: Spec deltas | v3 spec updates (canonicalization, exports, runtime metadata, nanopub mode, Parquet ordering) | Normative text \+ updated schemas \+ examples | Keep v0.1 compatibility where possible |
| P1: Reference tooling | Compiler updates \+ validators \+ export tooling | Reproducible builds, SHACL/ShEx hooks, WDQS export profile, Parquet ordering support | RDF canonicalization can be expensive; enforce resource limits |
| P2: Test corpus | Golden files: JCS hashes, RDF hashes, RDFC tests, export fixtures, query fixtures | Independent implementations pass the same vectors | Must include adversarial cases (canonicalization edge cases, blank-node stress tests) |
| P3: Migration & docs | v2→v3 migration guide \+ conformance checklist | Clear deprecations \+ compatibility matrix | Avoid breaking existing packs unexpectedly |

# **4\. Compatibility notes (v2 → v3)**

* Canonical JSON becomes normatively RFC 8785 (JCS). Existing v2 canonicalization rules remain acceptable if they are JCS-compatible.  
* New optional fields (rdf\_hash, canonicalization\_profile, SHACL report pointers, membership-filter build metadata, Parquet ordering policy) are additive.  
* Runtime pack readers should ignore unknown fields by default, but must fail closed on declared hashes/signatures that do not verify.  
* Nanopub/PROV-O mode is optional; packs without it remain valid.

# **Appendix A. Inputs used for this plan**

* Docv2Kristals.txt (v0.1/v2 draft spec text and schemas)  
* add1.txt–add5.txt (filters/bitmaps/parquet/subset builder/JSON-LD, SHACL/ShEx, WDQS export, truthy view, adaptivity loop)  
* add6.txt (literature-driven performance \+ provenance improvements: Parquet BRIs/Blooms, Roaring+Run, binary-fuse defaults, nanopubs/PROV-O, RDFC tests, JSON ingestion throughput)

# **Appendix B. add6 (verbatim reference)**

\#\# What this literature changes for Kristals (and what you already match)

\#\#\# 1\) Runtime Pack storage: push Parquet further (COTTAS → Kristals)

Your Runtime Pack is already explicitly “triple table \+ indexes \+ set algebra”, with Parquet recommended for the triples table . COTTAS shows how to get \*much more\* out of Parquet for RDF-like triple/quad tables:

\* \*\*Exploit Parquet BRIs (min/max “zone maps”)\*\* to prune row groups/pages during scans, especially if columns are \*well-clustered / ordered\* .

\* \*\*Exploit Parquet Bloom filters per column chunk\*\* to skip whole row groups when a value can’t be present (no false negatives; tuneable FPR) .

\* \*\*Treat “indexing” as “sorting the triple table”\*\*: COTTAS lists the classic SPO/SOP/PSO/POS/OSP/OPS orderings and explains the granularity trade-off via row-group sizing .

\*\*Concrete Kristals improvement\*\*

\* Add to \`manifest.json\` explicit \*sort order(s)\* used for the triples table and the \*\*row group size policy\*\*, because those directly control BRI/Bloom effectiveness (COTTAS calls this out) .

\* Consider \*\*optional Parquet-level Bloom filters on \`p\` and/or \`(p,o)\`\*\* (or on dictionary-encoded IDs), not just external membership filters. This complements your current “(p,o) → S-set” indexes .

\---

\#\#\# 2\) Index payloads: Roaring “run” optimization \+ row reordering

You already plan “compressed bitmaps for large sets” . The Roaring paper’s key operational insight is that \*\*data ordering changes container mix\*\*, and that \*\*Roaring+Run can shift storage heavily to run containers on sorted data\*\* .

\*\*Concrete Kristals improvement\*\*

\* When building S-sets, \*\*apply run optimization\*\* (Roaring+Run) and \*\*record container stats\*\* per index (optional) to drive future heuristics (the paper uses container stats as the lens) .

\* Consider \*\*row reordering / clustering\*\* upstream (triples table order, and/or ID assignment strategy) to increase runs, which helps both Roaring+Run and Parquet BRIs .

\---

\#\#\# 3\) Membership filters: make Binary Fuse the default (and standardize parameters)

Kristals already includes \*\*binary-fuse membership filters\*\* in the pack manifest example, with an explicit \`false\_positive\_rate\` , and the spec expects compatibility validation including FPR bounds .

Binary Fuse Filters provides the “why” and a performance target:

\* Binary fuse filters are designed to be \*\*closer to the theoretical lower bound\*\* than xor filters while keeping speed .

\* The paper also reports binary fuse can be \*\*\>2× faster to construct\*\* than xor filters in their experiments  and includes benchmark curves for construction/query time vs FPR .

\*\*Concrete Kristals improvement\*\*

\* Promote \`binary-fuse\` from “supported” to \*\*recommended default\*\*, and standardize:

  \* default FPRs per use case (e.g., \`claim\_id\` vs \`(s,p,o)\` hash),

  \* construction variant (3-wise vs 4-wise) as an explicit manifest parameter (their benchmarks distinguish them) .

\---

\#\#\# 4\) Provenance & trust: adopt Nanopublication structure \+ PROV-O terms

Kristals currently aims at deterministic IDs and offline execution; to make \*claims portable, attributable, and verifiable\*, nanopublications \+ PROV-O are the “standard Lego bricks”.

\*\*Nanopublications\*\*

\* A nanopublication is explicitly: \*\*assertion graph \+ provenance graph \+ publication info graph\*\*, linked from a head graph using \`np:hasAssertion\`, \`np:hasProvenance\`, \`np:hasPublicationInfo\` .

\* It also recommends \*\*Trusty URIs as integrity keys\*\* to enforce immutability / detect changes .

\*\*PROV-O\*\*

\* PROV-O gives the core classes (\*\*Entity / Activity / Agent\*\*) and the standard relations to form provenance chains (\`prov:used\`, \`prov:wasGeneratedBy\`, \`prov:wasDerivedFrom\`, attribution/association, timestamps) , with crisp definitions for \`prov:Entity\` .

\*\*Concrete Kristals improvement\*\*

\* For Kristal Exchange, define an optional “nanopub mode”:

  \* Assertion \= the resolved claim(s)

  \* Provenance \= \`prov:wasDerivedFrom\` source artifacts and \`prov:wasAttributedTo\` agents (fits the nanopub guidance) 

  \* Publication info \= compiler identity \+ build timestamp 

\* For Kristal IDs: consider aligning the Kristal/claim identifier with a \*\*trusty-URI-like hash commitment\*\* over the canonicalized dataset (nanopub integrity key pattern) .

\---

\#\#\# 5\) Canonicalization: use the RDFC-1.0 test suite as your compliance gate

You already talk about canonicalization/hashing and verifying pack compatibility & integrity . The RDFC-1.0 test suite is the practical “do we canonicalize blank nodes and datasets correctly?” gate, with many approved eval/map tests around blank-node structures and datasets .

\*\*Concrete Kristals improvement\*\*

\* Add to CI: run canonicalization on the RDFC test inputs and compare to expected outputs (eval tests) and/or expected identifier maps (map tests) .

\* Record in manifest the \*\*canonicalization profile/version\*\* used so hashes remain comparable across toolchains.

\---

\#\#\# 6\) JSON ingestion performance: “on-demand” parsing \+ simdjson-style two-stage

Kristals already keeps strings/JSON off the hot runtime path via dense integer IDs . The remaining pain point is \*\*Exchange ingestion/compilation\*\*.

\* The on-demand JSON literature argues for \*\*lazy materialization / only parse what you touch\*\*, rather than building full DOMs .

\* simdjson’s approach is built around a \*\*two-stage parse\*\* (structural scan then fast materialization), enabling very high throughput .

\*\*Concrete Kristals improvement\*\*

\* Specify a \*compiler requirement\*: Exchange readers should support \*\*field-skipping / lazy parsing\*\* for JSON(-LD) inputs, and report measured throughput in build metadata (so compilation cost becomes predictable, matching your runtime philosophy).

\---

\#\#\# 7\) Demand-driven subsets: make “subset recipes” first-class

You appear to want Kristals to package \*just enough KG\* for offline answering. The “demand-driven Wikidata subsets” work is essentially a formalization of that: build subsets from seeds and expand selectively with constraints .

\*\*Concrete Kristals improvement\*\*

\* Add a “subset construction recipe” section to Exchange/manifest:

  \* seeds (entities/properties)

  \* expansion rules (which edges, depth limits)

  \* filters (black/white lists)

  \* stopping criteria

\* This makes Kristals reproducible and comparable (“same recipe → same subset”), which pairs well with canonicalization+hashing.

\---

\#\# Summary of “already applied” vs “to add”

\*\*Already aligned in your Kristals doc\*\*

\* Triple table \+ (p,o)→S-set \+ set algebra model 

\* Parquet recommended for triples table 

\* Compressed bitmap/list encodings chosen by cardinality thresholds 

\* Binary-fuse membership filter in manifest example 

\*\*High-leverage additions\*\*

\* Treat Parquet ordering \+ BRIs \+ Bloom filters as \*core\* performance knobs (COTTAS) 

\* Standardize binary-fuse parameters and make it default (Binary Fuse Filters) 

\* Enable Roaring+Run and data clustering for better compression (Roaring) 

\* Add nanopub/PROV-O shaped provenance \+ integrity commitments (Nanopubs \+ PROV-O)  

\* Gate canonicalization on the W3C RDFC tests 

\* Make subset-construction recipes explicit and reproducible (demand-driven subsets)