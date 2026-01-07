# 00-overview/v2-to-v3-summary.md

## Status

Draft

## Date

2026-01-07 

## Purpose

Summarize the changes from Kristal v2 (draft) to Kristal v3, with emphasis on interoperability, determinism, and reproducible offline execution.

---

## What v3 is trying to achieve

v3 tightens the v2 draft into a more **interoperable** and **reproducible** standard, incorporating literature-driven improvements around Parquet-based RDF storage, Roaring+Run, binary-fuse filters, nanopublications/PROV-O, RDFC test gating, and high-throughput JSON ingestion patterns. 

### Key outcomes in v3

* **Normative canonicalization + hashing** that is portable across languages, with JSON-first identity and optional RDF-level hashing. 
* **Stronger RDF/JSON-LD interoperability**: stable JSON-LD 1.1 context/profile plus WDQS-compatible RDF export. 
* **Runtime Pack reproducibility**: fully specified membership-filter metadata, bitmap conventions, and Parquet ordering/row-group policy. 
* **More ergonomic offline query surface**: TPF-like pagination plus per-pattern cardinality metadata. 
* **First-class subset recipes + compiler build metadata** so offline subsets become reproducible artifacts. 
* **Portable provenance packaging** via an optional nanopublication/PROV-O shaped mode for Exchange exports. 

---

## Summary of major deltas (v2 → v3)

### 1) Canonicalization, identity, and integrity become portable and testable

* `canonical_json` becomes normatively **RFC 8785 (JCS)** to eliminate JSON ambiguity. 
* Signing/verification workflow is standardized (remove signatures → canonicalize → hash/verify), with explicit **fail-closed** failure modes. 
* Optional export integrity: **RDF Dataset Canonicalization (RDFC-1.0)** and an `rdf_hash` computed from canonical N-Quads. 
* CI gating: run selected RDFC tests and publish golden vectors; enforce resource limits for worst-case datasets. 

### 2) Export interoperability is formalized

* Define deterministic JSON-LD 1.1 profile and WDQS-compatible RDF export profile. 
* Optional “nanopub mode” export for Exchange with head graph linking assertion + provenance + publication info graphs; use PROV-O terms for portable attribution chains. 

### 3) Runtime Pack reproducibility and performance are specified (not implied)

* Parquet triples table: treat ordering as an index; record sort order(s) and row-group sizing policy (controls BRI effectiveness). 
* Optional Parquet-level Bloom filters to prune row groups during scans. 
* Membership filters: make **binary-fuse** the recommended default; standardize parameters (variant, seeds, bits/key) and record them; explicitly define what the filter gates and how false positives are pruned deterministically. 
* Roaring bitmaps: require run optimization (Roaring+Run) where beneficial; optionally record container statistics. 

### 4) Offline query surface gets a predictable “integration mode”

* Add TPF-like pagination for triple-pattern results and per-pattern cardinality (exact/approx) for tooling predictability. 
* Add optional compiled “truthy/best-rank” projection so offline defaults can match WDQS `wdt:` semantics. 
* Define portability profile for join1 caps (default cap + strict mode) to reduce cross-implementation divergence. 

### 5) Subset builder + compiler throughput become first-class

* “Subset recipes” become explicit: seeds, deterministic expansion rules, allow/deny lists, depth limits, stopping criteria, snapshot identifiers. 
* Compilers must support field-skipping / on-demand parsing for Exchange JSON/JSON-LD; recommend two-stage parsing; record build metrics (size, wall time, peak memory). 

---

## Compatibility and migration notes (v2 → v3)

* Canonical JSON becomes normatively **RFC 8785 (JCS)**; v2 canonicalization is acceptable only if JCS-compatible. 
* New fields (e.g., `rdf_hash`, `canonicalization_profile`, SHACL report pointers, membership-filter build metadata, Parquet ordering policy) are additive. 
* Runtime Pack readers should ignore unknown fields by default, **but must fail closed** on declared hashes/signatures that don’t verify. 
* Nanopub/PROV-O mode remains optional; packs without it remain valid. 

---

## Roadmap (summary)

* **P0: Spec deltas** — v3 spec updates: canonicalization, exports, runtime metadata, nanopub mode, Parquet ordering. 
* **P1: Reference tooling** — compiler updates + validators + export tooling; reproducible builds; SHACL/ShEx hooks; WDQS export profile; Parquet ordering support. 
* **P2: Test corpus** — golden files: JCS hashes, RDF hashes, RDFC tests, export fixtures, query fixtures; include adversarial canonicalization cases. 
* **P3: Migration & docs** — v2→v3 migration guide + conformance checklist; compatibility matrix. 

---

## Where this summary fits in the doc set

This overview is the “why/what changed” page. For details, see:

* `01-core-spec/` for normative identity + integrity rules
* `03-reproducibility/` for allowed policy sets and determinism requirements
* `05-profiles/` for optional RDFC / nanopub+PROV-O / SHACL+ShEx / TPF query modes
* `09-test-vectors/` for golden vectors and CI gating guidance
