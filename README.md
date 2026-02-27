# Kristal docs (v4 / v3 core)

This repository contains the documentation set for **Kristal v4**, defining the **v3 core artifact model** (Kristal Exchange + Runtime Pack) plus optional, standardized **profiles**.

Kristal is a portable, verifiable, **offline-executable** unit of encyclopedic knowledge, designed to be **Wikidata/Wikibase-aligned** and reproducible across toolchains.

## Start here

1) `00-overview/vision-and-scope.md`  
2) `00-overview/v2-to-v3-summary.md`  
3) `01-core-spec/` (normative core specification; start with the core “overview/spec” doc in this folder)

If you are implementing specific surfaces:
- IDs / canonicalization / hashing → `01-core-spec/ids-canonicalization-hashing.md`
- Signatures / trust / fail-closed rules → `01-core-spec/signatures-trust.md`
- Schemas (normative JSON Schemas) → `02-schemas/`
- Pack reproducibility + allowed policies → `03-reproducibility/`
- Offline query surface → `04-query/query-contract.md`
- Optional features (profiles) → `05-profiles/`
- Ecosystem contracts (Orgo/SenTient/Architect/Konnaxion) → `06-integration/`
- Security and multi-tenancy → `07-security/`
- Ops guidance (non-normative patterns) → `08-ops/`
- Golden vectors / fixtures → `09-test-vectors/`
- Worked examples → `10-examples/`

## What Kristal is (in one page)

Kristal is a compiled knowledge artifact (not free text), produced through a deterministic pipeline:

- **Claim-IR**: extractor output (schema-constrained; uncertainty + evidence)
- **SenTient**: resolution contract (ranked QID/PID candidates; preserve ambiguity)
- **Validation**: deterministic acceptance gate (“no compile on fail”)
- **Kristal Exchange**: canonical, content-addressed source of truth
- **Runtime Pack**: derived offline-executable index (no SPARQL; constrained query model)
- **Architect**: deterministic renderer (must trace outputs to claims/evidence)
- **Orgo**: operational control plane (workflow, auditing, distribution status)
- **Konnaxion**: distribution + offline UX (signed packs, caching, rollback rules)

## Conformance model

### v3 Core (required)

Implementations claiming **v3 core** conformance MUST (at minimum):

- Use **RFC 8785 (JCS)** for canonical JSON used in hashing.
- Define and implement the **exact hashed material** (including exclusions such as signatures).
- Enforce **fail-closed** behavior when hashes/signatures are declared but do not verify.
- Produce **reproducible Exchange/Runtime artifacts**: manifests record all build-affecting policies/parameters.
- Pass the **core test vectors** in `09-test-vectors/`.

### Profiles (optional)

Advanced capabilities are expressed as explicit profiles in `05-profiles/`.
Implementations MAY claim profile conformance individually (e.g., JSON-LD export, RDF/WDQS export, RDF integrity (RDFC), nanopub/PROV-O packaging, SHACL, ShEx, TPF-like pagination).

Profiles MUST:
- state requirements and limits,
- state what is hashed (if relevant),
- include conformance tests/fixtures.

## Repository structure

- `00-overview/` — scope, non-goals, deltas, ecosystem placement
- `01-core-spec/` — **normative core** (keep surface area small; strong defaults)
- `02-schemas/` — normative JSON Schemas (Claim-IR, Resolved Claim-IR, Validation Report, Exchange/Runtime manifests)
- `03-reproducibility/` — deterministic compilation rules + allowed portable policies + acceptance tests
- `04-query/` — offline query contract (constrained semantics; optional pagination profile)
- `05-profiles/` — optional standardized profiles (exports, integrity, provenance, validation, pagination, etc.)
- `06-integration/` — inter-system contracts (Orgo × SenTient × Architect × Konnaxion)
- `07-security/` — trust roots, verification, downgrade/rollback policy, multi-tenancy boundaries
- `08-ops/` — non-normative operational guidance (“senior architecture patterns” framing)
- `09-test-vectors/` — golden vectors for canonicalization/hashing (+ optional RDF integrity fixtures)
- `10-examples/` — worked examples for implementers

## Editing rules

- Normative language uses **MUST / SHOULD / MAY**.
- Keep the **core small**; push tunability into profiles or non-normative guidance.
- Any optional behavior MUST be expressed as a **profile**, not an undocumented extension.

## Versioning

Any change that affects hashes/IDs or deterministic build outputs requires:
- updated test vectors in `09-test-vectors/`,
- an explicit version bump in the relevant canonicalization/profile identifiers,
- and clear migration notes (when applicable).