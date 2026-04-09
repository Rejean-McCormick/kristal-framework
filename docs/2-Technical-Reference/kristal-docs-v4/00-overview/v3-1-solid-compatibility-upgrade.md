# Kristal v4.1 — Solid Compatibility Upgrade (Official Specification)

**Status:** Draft
**Version:** 3.1
**Date:** 2026-01-26
**Theme:** Solid-compatible surfaces, Kristal-native core.

---

## 1. Purpose

Kristal v4.1 introduces a Solid-inspired modular architecture for Kristal / Konnaxion without changing the v3 core artifact model (Exchange + Runtime Packs). v3.1 standardizes **composition (DI), storage abstraction, handler pipelines, representation conversion, patch semantics, initialization, and ACL-style authorization** so implementations can swap modules and add Solid-compatible interfaces while preserving offline determinism.

---

## 2. Goals

v3.1 MUST:

1. **Remain offline-correct and fail-closed** for declared integrity and trust. 
2. **Preserve deterministic identity and rebuild comparability** for Exchange and Runtime Packs.  
3. Provide a **config-composed runtime** where modules can be replaced without rewriting business logic.
4. Provide a **storage-agnostic store contract** so data location (USB/local/cloud) is not coupled to Orgo/Orchestrator logic.
5. Provide **pluggable handler chains** for request processing and pack activation. 
6. Provide **representation negotiation** via explicit export profiles (e.g., JSON-LD/RDF views) without changing canonical truth.
7. Provide **safe partial update semantics** (patch/diff) that do not corrupt content-addressing.

---

## 3. Non-goals (what we do NOT integrate from Solid)

Kristal v4.1 explicitly does **not** become a full Solid server:

1. **No full SPARQL 1.1 semantics** as the core query surface; the v3 query contract remains intentionally constrained (triple-pattern, deterministic paging/ordering). 
2. **No requirement that RDF is the canonical storage model.** RDF/JSON-LD are treated as export profiles; the canonical core remains the v3 Exchange model with content-addressed identity. 
3. **No URL-minted mutable identity model** (i.e., “resource identity = server URL”). Exchange identity remains `kristal_id` computed by hashing canonical JSON. 
4. **No network dependency for correctness** at activation time; trust roots must be pre-provisioned/cached so verification is offline-correct. 
5. **No “telemetry mutates facts” behavior.** Operational signals may be emitted, but must not directly mutate Exchange. 
6. **No deployment/ops patterns embedded into artifact conformance.** Operational resilience is separate from artifact contracts. 

---

## 4. Core invariants carried forward from v3

### 4.1 Canonicalization + identity

* v3.1 MUST keep explicit canonicalization profiles and versions, and MUST NOT claim conformance under an unspecified profile. 
* Exchange identity remains content-addressed:

`kristal_id = sha256(JCS(hash_target(exchange)))`
with `kristal_id` and signatures removed from the hash target. 

### 4.2 Reproducibility as an acceptance gate

* A core-conformant build MUST pass reproducibility acceptance criteria, including stable IDs and byte-stable pack outputs under identical inputs/policies. 
* Build determinism must avoid nondeterministic iteration/time/randomness in hashed/output bytes.  
* If validation fails, the compiler MUST NOT emit a conformance-claiming Exchange/Pack. 

---

## 5. v3.1 Architecture overview

v3.1 introduces a Solid-inspired modular stack:

1. **Composition Root (Config-Composed Runtime)**
2. **Store Contract (storage-agnostic persistence)**
3. **Handler Chains (Orgo request pipeline + Konnaxion activation pipeline)**
4. **Representation Layer (content negotiation + export profiles)**
5. **Identity + Patch Semantics (diff → new artifact)**
6. **Initialization + Authorization (bootstraps + ACL decision engine)**
7. **Distribution Controls (signed indexes, activation, rollback, downgrade prevention)**

---

## 6. Composition Root (Config-Composed Runtime)

### Requirements

* Implementations MUST build the runtime from declarative configuration (composition root) rather than hard-coded imports.
* The composition root MUST allow replacing:

  * store backends (USB/local/cloud/encrypted)
  * converters/export profiles
  * handler chain ordering
  * authorization strategies
  * Konnaxion channel policies and verification strategies

### Determinism requirement

* Build configuration MUST be part of the determinism surface and tracked (e.g., via `config_hash`). 

---

## 7. Store Contract (storage-agnostic)

### Requirements

* v3.1 MUST define a single store interface for reading/writing:

  * Exchange artifacts
  * Runtime Pack bundles + manifests
  * channel indices (e.g., `pack_index.json`)
  * caches and pins
* Store implementations MUST support a layout that enables **multiple installed versions per channel** and **atomic activation**. 
* Cache policies MUST be deterministic and defined (limits, eviction, pinning). 

---

## 8. Handler Chains

### 8.1 Orgo request pipeline

Orgo MUST process requests through an ordered chain of handlers (e.g., auth → negotiation → validation gate → action). Handler order MUST be configurable via the composition root.

### 8.2 Konnaxion activation pipeline

Activation MUST occur only if all mandatory checks succeed:

1. integrity verified (if declared)
2. manifest parses + schema validates
3. required files exist
4. runtime compatibility checks pass
5. activation is an **atomic switch** (no partial activation) 

---

## 9. Representation Layer (Solid-style negotiation via explicit export profiles)

### Requirements

* v3.1 SHOULD support client-driven content negotiation for *exports* (e.g., JSON-LD/RDF views) while keeping the canonical Exchange as the source of truth.
* Export behavior MUST be profile-driven and explicitly declared to preserve comparability and avoid silent divergence. 
* Query remains the v3 offline query contract (triple-pattern + deterministic ordering/paging), explicitly avoiding full SPARQL. 

---

## 10. Patch Semantics (safe partial updates)

### Requirements

* v3.1 MUST support partial updates without “overwrite the world” behavior.
* Patch application MUST be modeled as: **diff → produce new Exchange → re-canonicalize → compute new `kristal_id` → commit** (no in-place mutation of a content-addressed artifact). This preserves the `kristal_id` definition and reproducibility requirements. 

---

## 11. Initialization

### Requirements

On startup or “open Kristal,” implementations SHOULD run initializers that ensure:

* storage roots exist and are writable
* channel trust roots are pinned and available offline 
* channel policy configuration is present (activation rules, downgrade/rollback behavior)

---

## 12. Authorization (ACL-style decision engine)

### Requirements

* Authorization MUST be implemented as a standalone module that returns **allow/deny + reason**.
* Konnaxion MUST NOT mix trust roots or activation rules across channels unless explicitly configured. 

---

## 13. Distribution, Indexing, Downgrade Prevention, Rollback

### 13.1 Signed channel index (recommended)

Konnaxion SHOULD consume a signed `pack_index.json` containing:

* `channel_id`, `latest`, `pinned`, optional `minimum_allowed_version`, optional `revoked` 
  If present and signed, it MUST be verified using channel trust roots (fail-closed). 

### 13.2 Downgrade prevention (mandatory)

Konnaxion MUST implement deterministic downgrade prevention, including monotonic version rules and/or revocation-aware rules. 

### 13.3 Rollback (mandatory)

Konnaxion MUST support rollback modes such as pinned rollback and/or last-known-good rollback, and rollback must be deterministic given the same trigger sequence. 

---

## 14. Security and offline constraints

* Correct factual output MUST NOT require network calls. 
* If integrity is declared (hashes/signatures/kid), consumers MUST verify before activation and MUST fail closed on any verification failure. 
* Trust roots MUST be pinned per channel and must not be fetched over the network at activation time as a dependency for correctness. 

---

## 15. Conformance

A v3.1 implementation claiming conformance MUST:

1. Preserve v3 identity and canonicalization requirements.  
2. Pass v3 reproducibility acceptance criteria for Exchange and Runtime Packs. 
3. Enforce fail-closed verification where declared. 
4. Enforce deterministic activation (atomic switch) and deterministic rollback/downgrade rules. 

---

## 16. Migration notes (v3.0 → v3.1)

* v3.1 is a **major upgrade in architecture**, not a rewrite of the artifact model.
* Existing v3 Exchange/Pack artifacts remain valid; v3.1 primarily adds:

  * config-composed runtime wiring
  * store contract boundaries
  * handler-chain insertion points
  * negotiated export profiles
  * patch-as-new-artifact semantics
  * standardized distribution/index/rollback controls

---