Below is a stronger, more technical version you can use as a base spec.

---

# Kristal v5 — Normative Model for Collaborative Epistemic Compilation

**Status:** Draft
**Type:** Normative architecture and contract document
**Audience:** Architects, implementers, governance owners, pipeline owners, runtime/distribution operators
**Supersedes:** Kristal v4 pipeline assumptions where explicitly stated

## 1. Abstract

Kristal v5 is a **truth-plane contract system** for compiling **structured epistemic states** into immutable, portable, queryable, verifiable artifacts. Its defining change relative to Kristal v4 is that **compilation is no longer globally gated by validation success**. In v4, the normative pipeline required `Claim-IR` as the proposal boundary, `Resolved Claim-IR` as the resolution boundary, and deterministic validation as a hard acceptance gate; compilation had to stop if validation failed. kOA mirrored that same model in its architecture, component responsibilities, and Build Record invariants.   

Kristal v5 replaces that model with a two-stage trust architecture:

1. **Compile structured work into artifacts early.**
2. **Promote only some compiled artifacts to canonical status later.**

This change is motivated by collaborative knowledge construction. In that model, useful knowledge exists before final certainty exists. Drafts, objections, merges, partial validations, and confidence growth must be materializable as first-class artifacts rather than being blocked until they are already canonical.

## 2. Problem Statement

Kristal v4 assumed that the proposal boundary was primarily an **extraction boundary**. Extractors had to emit only Claim-IR; SenTient consumed Claim-IR and emitted Resolved Claim-IR; validation produced a deterministic report and blocked compilation on failure; Kristal then compiled only validated inputs into Exchange and Runtime Pack. kOA’s architecture described the same mandatory stage spine and the same hard acceptance behavior.   

That model becomes unsuitable when the system is opened to collaborative submission and iterative validation:

* `Claim-IR` becomes redundant if the initial object is already a structured human or hybrid draft rather than raw extractor output.
* “No compile on fail” prevents the system from preserving and distributing intermediate epistemic value.
* The old model conflates **artifact materialization** with **canonical acceptance**.
* The old model is optimized for certification pipelines, not for collaborative accumulation of epistemic capital.

Kristal v5 addresses that mismatch by moving the hard gate from **compile** to **promote**.

## 3. Scope

Kristal v5 remains the owner of the normative truth-plane concerns already assigned to Kristal in kOA: canonical truth representation, offline runtime representation, content-addressed identity, canonicalization, hashing, signatures/trust roots, manifests, and portable query semantics. kOA explicitly treats those as Kristal-owned external normative contracts rather than redefining them locally.   

Kristal v5 does **not** become a workflow engine, task system, or review UI. Workflow ordering, approvals, routing, audit, and operational escalation remain control-plane concerns owned by Orgo. Distribution, verification, activation, and rollback remain runtime/distribution concerns owned by Konnaxion.  

## 4. Design Goals

Kristal v5 has the following design goals:

* **Materialize uncertain work safely.**
* **Separate existence from trust level.**
* **Preserve deterministic identities and reproducibility.**
* **Retain fail-closed verification on trusted activation surfaces.**
* **Support collaborative review, merge, objection, and later promotion.**
* **Keep truth-plane contracts distinct from governance-plane and runtime-plane responsibilities.**

These goals preserve what Kristal v4 already treated as first-class: reproducibility, manifest integrity, deterministic contracts, and portable query/runtime behavior.  

## 5. Core Model Shift

### 5.1 v4 model

The v4 model can be summarized as:

`Ingest -> Extract -> Claim-IR -> Resolve -> Resolved Claim-IR -> Validate -> Compile -> Exchange + Runtime Pack -> Distribute/Activate`

In that model:

* Extractors must output only Claim-IR.
* SenTient must consume Claim-IR and emit deterministic Resolved Claim-IR.
* Validation must be deterministic and must block compilation on failure.
* Kristal compilation is defined over already validated inputs.
* Konnaxion verifies and activates fail-closed.   

### 5.2 v5 model

The v5 model is:

`Signal / Draft / Submission -> Structured Epistemic State -> Compile -> Working Artifact -> Review / Validation / Attestation / Merge -> Promote -> Canonical Artifact -> Trusted Distribution / Activation`

In that model:

* `Claim-IR` is no longer the mandatory universal input artifact.
* Compilation is permitted for non-canonical states.
* Validation remains deterministic where declared, but it no longer globally blocks compilation.
* Canonical status is granted by a separate promotion decision.
* Trusted activation remains stricter than mere compilation.

This is the central architectural difference of v5.

## 6. Normative Definitions

### 6.1 Structured Epistemic State

A **Structured Epistemic State** is the normative input unit for Kristal v5 compilation. It replaces Claim-IR as the universal required proposal artifact.

A Structured Epistemic State MUST be:

* schema-constrained;
* versioned or revisioned;
* content-addressable or canonically serializable for hashing;
* provenance-bearing;
* scope-aware where tenant or environment boundaries apply;
* explicit about uncertainty, status, and lineage.

It MAY originate from:

* automated extraction;
* human submission;
* collaborative editing;
* merge or deduplication workflows;
* corrective updates;
* review objections;
* hybrid human/LLM authoring.

### 6.2 Working Artifact

A **Working Artifact** is a compiled Kristal artifact representing a structured epistemic state that is **not yet canonical**.

A Working Artifact MUST be:

* immutable once emitted;
* content-addressed;
* queryable through the declared Kristal query surface;
* explicit about its trust class;
* distinguishable from canonical artifacts in both metadata and distribution policy.

A Working Artifact MAY contain:

* partial assertions;
* disputed assertions;
* unresolved ambiguity;
* low-confidence assertions;
* review-pending content.

### 6.3 Canonical Artifact

A **Canonical Artifact** is a compiled artifact that has been **promoted** under the applicable governance and trust policy.

Canonical status is not a serialization property. It is a **trust-state property** granted by promotion.

### 6.4 Promotion

**Promotion** is the operation that changes a compiled artifact from working status to canonical status for one or more trust surfaces.

Promotion MUST be policy-controlled and auditable.

## 7. Artifact Taxonomy

Kristal v5 SHOULD define at least the following artifact classes.

### 7.1 Epistemic State Document

The input artifact. It SHOULD include at minimum:

* `artifact_type`
* `artifact_version`
* `artifact_id` or canonical hash target
* `tenant_id` / `scope` if applicable
* `revision_id`
* `created_at`
* `created_by`
* `derived_from[]`
* `merged_from[]`
* `supersedes`
* `status`
* `certainty`
* `assertions[]`
* `provenance[]`
* `review_refs[]`
* `policy_refs[]`

### 7.2 Working Exchange

The compiled working truth artifact.

It SHOULD include:

* stable identity
* compiler identity
* config hash
* source epistemic state reference
* policy selections
* trust tier = `working`
* declared query compatibility
* declared export/profile compatibility

### 7.3 Promotion / Review Bundle

The artifact family that records why a working artifact is or is not promotable.

It SHOULD include:

* validation outputs
* review outputs
* attestation refs
* quorum status
* promotion decision
* reason codes
* applicable policy refs
* trust-root or signer context where applicable

### 7.4 Canonical Exchange

The promoted truth artifact.

It MUST be distinguishable from a Working Exchange and MUST retain traceability to the working or source state from which it was produced.

### 7.5 Runtime Pack

The offline query/runtime artifact persists in v5, but it MUST declare its source trust tier. A Runtime Pack derived from a Working Exchange is not equivalent to one derived from a Canonical Exchange.

That distinction matters operationally because Konnaxion already enforces strong verification, compatibility, and activation semantics on runtime artifacts.  

## 8. Required Semantics

### 8.1 Compilation semantics

In Kristal v5:

* Compilation MUST be deterministic within the declared reproducibility surface.
* Compilation MUST NOT imply canonical acceptance.
* Compilation MAY succeed even when promotion eligibility is not satisfied.
* Compilation output MUST encode its trust class explicitly.

This preserves the v4 reproducibility principle: same declared inputs, compiler version, config hash, and policy selections must reproduce the same identities and declared hashes.  

### 8.2 Validation semantics

Validation in v5 remains important, but its role changes.

Validation MUST still be:

* structured;
* machine-readable;
* deterministically replayable where the validation profile declares determinism;
* stable in its codes and reason model.

However:

* Validation MUST NOT be treated as a universal compile blocker.
* Validation MAY block promotion.
* Validation MAY block publication on selected channels.
* Validation MAY block activation on selected channels.

This is the exact place where v5 diverges from the v4 contract, which explicitly required compilation to stop when validation failed.  

### 8.3 Promotion semantics

Promotion is the new hard gate.

A promotion decision SHOULD evaluate, at minimum:

* schema conformance;
* provenance sufficiency;
* identity/hash integrity;
* fatal validation findings;
* policy compliance;
* required attestations or approvals;
* trust-surface compatibility;
* reproducibility evidence where required.

If promotion fails, canonical publication MUST NOT proceed.

### 8.4 Query semantics

Kristal v5 SHOULD preserve the portable constrained query surface already present in Kristal v4. Runtime-capable artifacts must remain queryable offline according to the declared query contract and ordering policy. Working artifacts and canonical artifacts may expose the same core query interface, but they MUST differ in trust metadata and policy affordances. 

## 9. Status Model

Every v5 compiled artifact MUST declare a machine-readable trust/state classification. A minimal status model SHOULD include:

* `draft`
* `under_review`
* `partial`
* `disputed`
* `working_accepted`
* `promoted`
* `canonical`
* `revoked`

The exact token set may evolve, but the distinction between **working** and **canonical** MUST be explicit and non-derivable only by convention.

## 10. Determinism and Reproducibility

Kristal v5 does not weaken determinism.

The following remain mandatory within the declared reproducibility surface:

* schema-valid manifests;
* explicit compiler identity;
* explicit configuration hash;
* explicit policy selections;
* stable identity generation;
* stable payload hashes;
* deterministic sorting and normalization where part of the identity or validation surface.

Kristal v4 already treated reproducibility as first-class and required identical rebuilds under identical snapshots, compiler version, config hash, and policy selections. v5 inherits that requirement.  

## 11. Security and Trust Model

Kristal v5 separates **artifact existence** from **artifact trust**, but it does not weaken trust enforcement where trust matters.

### 11.1 Working surfaces

Working artifacts MAY exist without being promotion-eligible. They MAY be distributed to draft, review, sandbox, or internal collaboration surfaces according to policy.

### 11.2 Trusted surfaces

Trusted channels MUST remain fail-closed. If a channel requires signatures, hashes, trust-root validation, compatibility checks, or promotion status, those requirements MUST be enforced before activation or trusted publication. This is consistent with the existing Konnaxion contract, which already requires fail-closed verification, atomic activation, and deterministic rollback behavior.  

### 11.3 Multi-tenant isolation

If collaborative review or resolution occurs in multi-tenant environments:

* tenant-specific context MUST remain isolated;
* evidence pointers MUST NOT leak across tenants;
* output metadata MUST preserve tenant scoping where applicable.

These protections already exist in the SenTient boundary guidance and remain required in v5. 

## 12. Responsibilities Across the Stack

The v5 split of responsibilities is:

* **Orgo** owns workflow, review process, approvals, audit, routing, operational policies, and lifecycle control.
* **Kristal** owns compiled truth artifacts, identities, manifests, query contracts, and promotion-capable truth representations.
* **Konnaxion** owns distribution, verification, activation, and rollback.
* **Collaborative/social layers** may own voting, weighting, diversity, or reputation signals, but they do not redefine Kristal artifact identity rules.

This remains aligned with the existing kOA component map, which already places Orgo in the control plane, Kristal in the truth plane, and Konnaxion in distribution/runtime.  

## 13. Impact on kOA Operational Records

Kristal v5 requires corresponding changes in kOA-owned operational artifacts.

The current Build Record assumes a v4 pipeline in which validation gates compilation and failed validation prevents Exchange/Runtime Pack outputs from appearing as successful build outputs. That invariant must be replaced. 

A v5-aware Build Record SHOULD distinguish at least:

* `compile_status`
* `validation_status`
* `review_status`
* `promotion_status`
* `working_outputs[]`
* `canonical_outputs[]`
* `publication_status`
* `activation_status`
* `non_promotion_reason_codes[]`

This is necessary because in v5, a build can succeed as a working compilation even if canonical promotion is denied.

## 14. Migration Consequences

Kristal v5 is a **major-version change** because it alters two central v4 invariants:

1. `Claim-IR` is no longer the universal mandatory proposal boundary.
2. Validation is no longer the universal compile gate.

That means migration will require:

* revised Orgo integration contracts;
* revised producer interfaces;
* revised validation/review artifact contracts;
* revised build and release records;
* revised conformance criteria;
* explicit coexistence or conversion rules for v4 artifacts.

## 15. Non-Goals

Kristal v5 does not aim to:

* replace Orgo as the workflow/control plane;
* turn every compiled artifact into canonical truth;
* allow best-effort activation on trusted channels;
* remove signatures, trust roots, manifests, or content-addressed identity;
* collapse governance, compilation, and runtime into one layer.

## 16. Normative Summary

Kristal v5 is defined by the following rules:

* A structured epistemic state MAY be compiled before it is canonical.
* Compilation MUST be deterministic within the declared reproducibility surface.
* Validation MUST remain structured and machine-readable.
* Validation MUST NOT universally block compilation.
* Promotion MUST be the hard gate for canonical status.
* Trusted publication and activation MUST remain fail-closed where policy requires it.
* Working and canonical artifacts MUST be explicitly distinguishable.
* Multi-tenant provenance and evidence boundaries MUST remain protected.

## 17. One-Sentence Definition

> **Kristal v5 is a deterministic artifact system that compiles structured epistemic states into portable truth artifacts, while moving the hard trust boundary from compile-time validation to governed promotion and trusted activation.**

If you want, I can convert this into a **formal RFC-style spec** next, with explicit `MUST / SHOULD / MAY`, artifact field sketches, and a matching file path such as `docs/00-overview/what-is-kristal-v5.md`.
