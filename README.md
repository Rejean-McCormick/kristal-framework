# Kristal docs (v5)

This repository contains the documentation set for **Kristal v5**, a normative framework for **collaborative epistemic compilation**.

Kristal v5 defines how **structured epistemic states** are compiled into immutable, portable, queryable, verifiable artifacts, while separating:

- **compilation** from **canonization**
- **working artifacts** from **canonical artifacts**
- **review/promotion** from **trusted distribution and activation**

Kristal remains designed for **portable**, **verifiable**, **offline-capable** knowledge operation across toolchains and environments.

---

## Start here

1. `00-overview/what-is-kristal-v5.md`  
2. `00-overview/vision-and-scope.md`  
3. `00-overview/v4-to-v5-summary.md`  
4. `01-core-spec/` (normative core specification; start with the core overview/spec document in this folder)

If you are implementing specific surfaces:

- Artifact model / lifecycle / trust classes → `01-core-spec/`
- IDs / canonicalization / hashing → `01-core-spec/ids-canonicalization-hashing.md`
- Promotion / signatures / trust / fail-closed rules → `01-core-spec/signatures-trust.md`
- Schemas (normative JSON Schemas) → `02-schemas/`
- Reproducibility + build surfaces → `03-reproducibility/`
- Offline query surface → `04-query/query-contract.md`
- Optional features (profiles) → `05-profiles/`
- Ecosystem contracts (Orgo / SenTient / Architect / Konnaxion) → `06-integration/`
- Security and multi-tenancy → `07-security/`
- Ops guidance (non-normative patterns) → `08-ops/`
- Golden vectors / fixtures → `09-test-vectors/`
- Worked examples → `10-examples/`

---

## What Kristal v5 is

Kristal v5 is a deterministic artifact system for **compiling structured epistemic work**.

It is built around the following model:

- **Structured Epistemic State**: the normative input unit for compilation
- **Working Exchange**: compiled, immutable, queryable, non-canonical truth artifact
- **Review / Promotion Bundle**: validation, attestation, policy, quorum, and promotion decision surface
- **Canonical Exchange**: promoted truth artifact accepted for strong trust surfaces
- **Runtime Pack**: derived offline-capable query/runtime artifact
- **Architect**: deterministic rendering surface; outputs must remain traceable to compiled knowledge
- **Orgo**: control plane for workflow, approvals, audit, routing, and operational governance
- **Konnaxion**: distribution, verification, activation, caching, rollback, and trust-surface enforcement

---

## Core architectural change in v5

Kristal v5 replaces the v4 assumption that only fully validated inputs may be compiled.

### v4 model
- `Claim-IR` was the universal proposal boundary
- validation acted as a hard acceptance gate
- compilation stopped on validation failure

### v5 model
- `Claim-IR` is no longer the mandatory universal input artifact
- structured epistemic states may be compiled before canonization
- validation no longer universally blocks compilation
- **promotion** is the hard gate for canonical status
- trusted publication and activation remain fail-closed where policy requires it

In short:

> **Kristal v5 compiles early and promotes later.**

This allows drafts, review states, objections, partial certainty, and collaborative refinement to exist as first-class artifacts without being confused with canonical truth.

---

## Artifact classes

Kristal v5 distinguishes at least the following artifact classes:

### 1. Structured Epistemic State
A schema-constrained, versioned, provenance-bearing assertional state suitable for compilation.

Typical contents:
- identity / revision
- scope / tenant metadata where applicable
- assertions
- certainty / uncertainty metadata
- provenance references
- lineage (`derived_from`, `merged_from`, `supersedes`)
- review / objection / attestation references

### 2. Working Exchange
A compiled artifact representing a **non-canonical** epistemic state.

Properties:
- immutable
- content-addressed
- queryable
- portable
- reproducible within its declared surface
- explicitly marked as `working`

### 3. Review / Promotion Bundle
The artifact family that records validation, policy, attestation, quorum, and promotion decisions.

### 4. Canonical Exchange
A promoted artifact recognized as canonical for one or more trust surfaces.

### 5. Runtime Pack
The offline-capable runtime/query representation derived from a Working Exchange or Canonical Exchange.

A Runtime Pack MUST declare its source trust tier.

---

## Trust model

Kristal v5 separates **artifact existence** from **artifact trust level**.

### Working trust surfaces
These may accept non-canonical compiled artifacts for:
- collaborative review
- merge / deduplication
- internal drafting
- semantic discovery
- sandbox / staging workflows

### Canonical trust surfaces
These accept only promoted artifacts and retain strict verification requirements:
- schema verification
- hash verification
- signature verification
- trust-root enforcement
- compatibility checks
- downgrade prevention
- atomic activation
- deterministic rollback

Kristal v5 relaxes compilation. It does **not** relax trust enforcement on trusted surfaces.

---

## Conformance model

### v5 Core (required)

Implementations claiming **Kristal v5 core** conformance MUST, at minimum:

- define a valid **Structured Epistemic State** input surface
- distinguish **working** and **canonical** artifact classes explicitly
- support deterministic compilation within the declared reproducibility surface
- use the declared canonicalization and hashing rules for identity-bearing artifacts
- enforce **fail-closed** behavior where hashes, signatures, trust roots, or promotion requirements are declared as mandatory
- treat **promotion** as distinct from **compilation**
- preserve traceability from compiled artifacts back to provenance-bearing source state(s)
- produce reproducible manifests recording build-affecting configuration, policy, and compiler identity
- pass the **core test vectors** in `09-test-vectors/`

### Profiles (optional)

Advanced capabilities are expressed as explicit profiles in `05-profiles/`.

Implementations MAY claim profile conformance individually, including profiles such as:
- export formats
- provenance packaging
- integrity extensions
- review / attestation models
- pagination / query extensions
- validation frameworks

Profiles MUST:
- state requirements and limits
- state what is hashed, signed, or identity-bearing
- state whether they affect reproducibility surfaces
- include conformance tests or fixtures

---

## Repository structure

- `00-overview/` — scope, non-goals, version deltas, ecosystem placement, migration notes
- `01-core-spec/` — **normative core** (artifact model, lifecycle, trust model, promotion semantics, determinism)
- `02-schemas/` — normative JSON Schemas for v5 artifacts
- `03-reproducibility/` — deterministic compilation rules, identity surfaces, acceptance tests
- `04-query/` — offline query contract and related constraints
- `05-profiles/` — optional standardized profiles
- `06-integration/` — inter-system contracts (Orgo × SenTient × Architect × Konnaxion)
- `07-security/` — trust roots, verification, promotion trust rules, downgrade/rollback policy, multi-tenancy boundaries
- `08-ops/` — non-normative operational guidance
- `09-test-vectors/` — golden vectors for canonicalization, hashing, manifests, and other identity-bearing surfaces
- `10-examples/` — worked examples for implementers

---

## Editing rules

- Normative language uses **MUST / SHOULD / MAY**.
- Keep the **core small and explicit**.
- Do not hide optional behavior inside undocumented extensions.
- Any optional behavior that affects identity, trust, reproducibility, query semantics, or distribution MUST be expressed as a **profile** or an explicitly versioned core rule.
- Do not conflate:
  - compile with promote
  - working with canonical
  - workflow policy with truth-plane identity rules

---

## Versioning

Any change that affects hashes, IDs, deterministic outputs, promotion semantics, or trust-surface behavior requires:

- updated test vectors in `09-test-vectors/`
- an explicit version bump in the relevant canonicalization, schema, profile, or artifact identifiers
- clear migration notes
- compatibility guidance where applicable

Major-version changes are required for changes that alter:
- core artifact classes
- identity-bearing canonicalization or hashing rules
- compile vs promote semantics
- trust-class semantics
- runtime compatibility guarantees

---

## One-sentence definition

> Kristal v5 is a deterministic truth-artifact framework that compiles structured epistemic states into portable and verifiable artifacts, while moving the hard trust boundary from compile-time validation to governed promotion and trusted activation.