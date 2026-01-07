# Kristal v3 Documentation

This repository contains the v3 documentation set for **Kristal**: the portable, verifiable, offline-executable unit of encyclopedic knowledge (Exchange + Runtime Pack), designed to be Wikidata/Wikibase-aligned and reproducible across toolchains.

## How to use this repo

Start here:
1) `00-overview/vision-and-scope.md`
2) `00-overview/v2-to-v3-summary.md`
3) `01-core-spec/kristal-v3-core-spec.md`

If you are implementing:
- IDs / hashing / signing → `01-core-spec/ids-canonicalization-hashing.md`, `01-core-spec/signatures-trust.md`
- Schemas → `02-schemas/`
- Pack reproducibility → `03-reproducibility/`
- Offline query surface → `04-query/query-contract.md`
- Optional features → `05-profiles/`
- Ecosystem integration (Orgo/SenTient/Architect/Konnaxion) → `06-integration/`
- Security and multi-tenancy → `07-security/`
- Ops guidance (non-normative patterns) → `08-ops/`

## Repository structure

- `00-overview/`  
  High-level scope, non-goals, and the v2 → v3 delta summary.

- `01-core-spec/`  
  **Normative core**. Keep surface area small. Strong defaults. Everything else is via profiles.

- `02-schemas/`  
  **Normative JSON Schemas** for Claim-IR, Resolved Claim-IR, Validation Report, Exchange Manifest, Runtime Pack Manifest.

- `03-reproducibility/`  
  Deterministic compilation rules and **allowed policies** (enumerated ordering/row-group/filter policies) plus acceptance tests.

- `04-query/`  
  Offline query contract (constrained semantics; optional TPF-like pagination profile).

- `05-profiles/`  
  Optional standardized profiles (JSON-LD export, RDF/WDQS export, RDFC integrity, nanopub/PROV-O, SHACL, ShEx, query pagination).

- `06-integration/`  
  Inter-system contracts for **Orgo × SenTient × Architect × Konnaxion**.

- `07-security/`  
  Trust roots, signature verification, downgrade/rollback policy, multi-tenancy boundaries.

- `08-ops/`  
  Non-normative operational guidance using “senior architecture patterns” framing (failure paths, correlation IDs, canary/blue-green).

- `09-test-vectors/`  
  Golden vectors for canonicalization/hashing and (optional) RDF integrity fixtures.

- `10-examples/`  
  Worked examples for implementers.

## Conformance model

### v3 Core (required)
Implementations claiming **Kristal v3 core** conformance MUST:
- Use **RFC 8785 (JCS)** for canonical JSON used in hashing.
- Define **exact hashed material** (including exclusions such as signatures) and implement it identically.
- Enforce **fail-closed** behavior when hashes/signatures are declared but do not verify.
- Produce **reproducible Exchange/Runtime artifacts**: manifests record all build-affecting policies/parameters.
- Pass the **core test vectors** in `09-test-vectors/`.

### Profiles (optional)
Advanced features are expressed as explicit profiles in `05-profiles/`. Implementations MAY claim profile conformance individually (e.g., RDF Integrity (RDFC), nanopub/PROV-O, SHACL, ShEx, TPF-like pagination).

## Ecosystem placement (summary)
- **Claim-IR**: the only direct extractor output (strict schema; uncertainty + evidence).
- **SenTient**: resolution contract (ranked QID/PID candidates; preserve unresolved ambiguity).
- **Validation**: deterministic acceptance gate (“no compile on fail”).
- **Kristal Exchange**: canonical, content-addressed source of truth.
- **Runtime Pack**: derived offline-executable index (no SPARQL; constrained query model).
- **Architect**: deterministic renderer after validation; must trace outputs to claim/evidence.
- **Orgo**: operational control plane (workflow, auditing, distribution status).
- **Konnaxion**: distribution + offline UX (signed packs, caching, rollback rules).

## Editing rules for this repo

- Normative language uses **MUST / SHOULD / MAY**.
- Each profile doc must clearly state:
  - requirements,
  - what is hashed (if relevant),
  - limits (timeouts/resource bounds where needed),
  - conformance tests and fixtures.

- Keep the **core small**; push tunability into profiles or non-normative guidance.

## Versioning
- This repo documents **Kristal v3**. v2 compatibility notes belong in `00-overview/v2-to-v3-summary.md`.
- Any change that affects hashes/IDs or deterministic build outputs requires:
  - updated test vectors in `09-test-vectors/`,
  - explicit version bump in the relevant canonicalization/profile identifiers.
