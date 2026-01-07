# Vision and scope (Kristal v3)

Kristal is the portable, verifiable, offline-executable unit of encyclopedic knowledge in the ecosystem. A Kristal is **not a document** and not free text. It is a **compiled knowledge artifact** designed to be **Wikidata/Wikibase-aligned**, **traceable**, **AI-ready**, and **executable offline**.

Kristal v3 tightens interoperability and reproducibility by making canonicalization + hashing unambiguous, defining strict pipeline contracts, and requiring deterministic build manifests—while keeping the normative core small and expressing advanced capabilities as explicit profiles.

## Goals

Kristal v3 aims to:

1. **Interoperable identity**
   - Identical content yields identical IDs across languages and toolchains.
   - Hashing and signature verification behavior is fully specified.

2. **Deterministic compilation**
   - Exchange and Runtime Pack generation is reproducible.
   - Build-affecting policies and parameters are recorded in manifests.

3. **Strict pipeline boundaries**
   - Extractors produce **Claim-IR only** (schema-constrained proposals with uncertainty and evidence).
   - SenTient performs resolution without forced disambiguation.
   - Validation is a deterministic acceptance gate (“no compile on fail”).
   - Architect renders deterministic outputs after validation and cannot introduce new facts.

4. **Offline execution**
   - Runtime Packs are executable offline and do not depend on SPARQL endpoints, network access, or LLMs.
   - Query semantics are intentionally constrained to remain predictable and portable.

5. **Standards-aligned exports**
   - Define stable export profiles (JSON-LD and RDF/WDQS-aligned projections).
   - Provide optional integrity and provenance packaging profiles for high-assurance contexts.

## Non-goals

Kristal v3 does **not** attempt to:

- Provide full SPARQL semantics in Runtime Packs.
- Encode operational/deployment patterns as first-class objects inside Exchange or Runtime Pack schemas.
- Guarantee identical performance across implementations (only identical outputs for deterministic modes).
- Replace Orgo’s operational logging/auditing system (Kristal records knowledge and build metadata; Orgo records workflow and governance).

## Core model and artifacts

Kristal exists as: **standard + compilation pipeline + runtime pack**.

### Core artifacts (release outputs)
Each Kristal release produces two primary artifacts:

1. **Kristal Exchange**
   - Canonical, content-addressed, auditable source of truth for validated knowledge.
   - Designed to be mergeable and comparable across toolchains.

2. **Kristal Runtime Pack**
   - Derived, offline-executable indexed form for constrained queries.
   - Optimized for offline distribution and predictable execution.

### Pipeline stages (conceptual)
1. **Ingest**: documents/web/PDF/datasets/signals
2. **Claim-IR**: extractor outputs schema-constrained proposals (uncertainty + evidence)
3. **Resolution (SenTient)**: surfaces → ranked QIDs/PIDs; normalize values; preserve unresolved ambiguity
4. **Validation**: deterministic acceptance gate; if validation fails, compilation MUST stop
5. **Exchange finalize**: canonical + signed + content-addressed
6. **Runtime Pack compile**: deterministic, manifest-recorded policies
7. **Render (Architect)**: deterministic generation after validation; must trace to claims/evidence

## Normative core vs profiles

Kristal v3 is structured as:
- **Core (normative, required):** minimal determinism surface area with strong defaults
- **Profiles (optional, standardized):** advanced features (integrity, provenance, richer exports, pagination)

### v3 Core includes (high level)
- RFC 8785 (JCS) canonicalization for hashed JSON objects
- Explicit hashing material and exclusions (e.g., signatures excluded from hashed payload)
- Fail-closed verification when declared hashes/signatures do not verify
- Deterministic build requirements and reproducible manifests
- Core schemas and core test vectors

### Profiles include (examples)
- JSON-LD export profile
- RDF/WDQS-aligned export profile
- RDF Integrity profile (RDFC + resource limits + CI gating)
- Provenance packaging profile (nanopub + PROV-O)
- Validation profiles (SHACL, ShEx)
- Query pagination profile (TPF-like semantics)

## Ecosystem integration placement (Orgo × SenTient × Architect × Konnaxion)

- **Orgo**: operational control plane for Kristal workflows (ingest → resolve → validate → publish), including audits and distribution status.
- **SenTient**: resolver/reconciler implementing the resolution contract.
- **Architect**: deterministic renderer consuming validated claims/query results, producing traceable outputs without adding facts.
- **Konnaxion**: distribution and offline UX layer for Runtime Packs (verification, caching, versioning, rollback/downgrade rules).

Operational patterns (circuit breakers, DLQs, CQRS framing, canary/blue-green rollout, structured logs + correlation IDs) are used as **non-normative guidance** for the build/distribution system and are not embedded into Kristal artifact schemas.

## Document map

- Overview and deltas: `00-overview/`
- Normative core: `01-core-spec/`
- Schemas: `02-schemas/`
- Reproducibility: `03-reproducibility/`
- Query: `04-query/`
- Profiles: `05-profiles/`
- Integration contracts: `06-integration/`
- Security and tenancy: `07-security/`
- Operational guidance (non-normative): `08-ops/`
- Test vectors: `09-test-vectors/`
- Examples: `10-examples/`
