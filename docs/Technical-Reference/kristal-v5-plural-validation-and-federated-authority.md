# Kristal v5 — Plural Validation, Federation, and Authority Channels

**Status:** Draft  
**Type:** Normative architecture note  
**Audience:** Architects, implementers, governance owners, validator operators, federation/runtime operators  
**Proposed path:** `kristal-docs-v5/00-overview/plural-validation-and-federated-authority.md`  
**Relates to:** Kristal v4 sharding/federation, authority registry, reproducibility, validation reports, Runtime Pack distribution, and the Kristal v5 collaborative compilation upgrade.

---

## 1. Purpose

This document defines how Kristal v5 treats validation, authority, federation, and divergent corpora.

Kristal v5 does not assume that all compiled knowledge is already true, final, expert-approved, or globally accepted. It allows hypotheses, disputed claims, incomplete assertions, creative corpora, fictional worlds, mythological structures, research proposals, institutional knowledge, and validated reference corpora to exist in the same ecosystem without being confused.

The core invariant is:

> A Kristal MAY contain uncertain, disputed, fictional, mythological, speculative, incomplete, or erroneous assertions.  
> A Kristal MUST NOT present those assertions as validated truth except within an explicitly declared authority channel, scope, and validation policy that supports that status.

This document replaces the global “validation gates compilation” framing from Kristal v4 with a plural validation model for Kristal v5.

---

## 2. Background: v4 assumptions being revised

Kristal v4 defines Kristal as a standard, compilation pipeline, and runtime pack. Its default flow is:

`Claim-IR -> SenTient resolution -> deterministic validation -> Kristal Exchange -> Runtime Pack`

In that model, validation is a deterministic acceptance gate, and failed validation prevents compilation. This is appropriate for certification-style pipelines, but too restrictive for collaborative epistemic construction.

Kristal v5 keeps the v4 strengths:

* deterministic identities;
* reproducible builds;
* content addressing;
* signatures and trust roots;
* explicit evidence and provenance;
* offline-verifiable Runtime Packs;
* scoped sharding and federation;
* authority registries and revocation-aware verification.

But it changes the epistemic lifecycle:

`Structured Epistemic State -> Compile -> Working Artifact -> Review / Validation / Attestation / Federation -> Recognized Reference Artifact`

Compilation is no longer equivalent to truth acceptance. Recognition is scoped.

---

## 3. Terminology

### 3.1 Kristal

A **Kristal** is a compiled, portable, verifiable, queryable knowledge artifact or artifact family.

A Kristal is not intrinsically a “scientific Kristal”, “mythology Kristal”, “policy Kristal”, or “fiction Kristal”. Those classifications are determined by metadata, scope, epistemic mode, authority channel, validation policy, and federation context.

### 3.2 Assertion

An **assertion** is a claim-like unit inside a Kristal. It may represent a factual statement, hypothesis, disputed claim, fictional claim, mythological claim, policy claim, system description, cultural claim, or other structured statement.

Assertions SHOULD carry explicit provenance, certainty, and status metadata.

### 3.3 Shard

A **shard** is a scoped Kristal artifact or Exchange unit representing a portion of a larger corpus. Shards may be scoped by domain, subdomain, time window, tenant, environment, source corpus, authority channel, or other declared policy dimensions.

### 3.4 Federation

A **federation** is a deterministic composition of multiple shards or Kristals under a federation root. Federation allows multiple sources and authorities to coexist without rewriting or silently merging their underlying claims.

Federation MUST preserve lineage, source identity, shard identity, authority identity, and conflict visibility.

### 3.5 Authority channel

An **authority channel** is a declared source of recognition, validation, review, or approval.

Examples include:

* an individual researcher;
* a laboratory;
* a professional association;
* a standards body;
* a government agency;
* a company describing its own systems;
* an expert consortium;
* a global institution;
* an AI-assisted validator operating under a declared policy.

An authority channel does not own truth globally. It can only recognize claims, shards, or corpora within a declared scope and validation policy.

### 3.6 Recognition

**Recognition** is the operation by which an authority channel accepts an artifact, shard, assertion, source corpus, or validation report under a declared policy.

Kristal v5 SHOULD prefer the terms `recognized`, `reference`, `validated`, `authority_approved`, or `trusted_for_scope` over unqualified `canonical`.

If the term `canonical` is retained for compatibility, it MUST be interpreted as “canonical within a declared authority channel and scope,” never as universal truth.

---

## 4. Scope and epistemic mode

Every Kristal v5 artifact, shard, or assertion SHOULD declare enough metadata for consumers to understand how it should be interpreted.

At minimum, v5 metadata SHOULD support:

```json
{
  "scope": {
    "domain": "science | medicine | education | mythology | fiction | policy | system-description | culture | other",
    "subdomain": "optional",
    "time_window": "optional"
  },
  "epistemic_mode": "factual | hypothesis | disputed | fictional | mythological | doctrinal | experimental | operational | imported",
  "authority_channel": "authority:<id>",
  "validation_policy": "policy:<id>",
  "recognition_status": "unreviewed | working | reviewed | validated | rejected | disputed | reference",
  "certainty": "hypothesis | claimed | sourced | corroborated | reviewed | validated | reference"
}
```

The exact vocabulary MAY evolve, but the distinction between artifact existence, epistemic mode, authority channel, and recognition status MUST remain explicit.

---

## 5. Validation levels

Kristal v5 recognizes multiple validation levels.

### 5.1 Artifact-level validation

Artifact-level validation answers:

* Is the artifact schema-conformant?
* Are identities and hashes correct?
* Are signatures and trust roots valid where required?
* Is the artifact reproducible under its declared policies?
* Is its provenance and lineage preserved?
* Is it safe to distribute or execute under the declared runtime policy?

Artifact-level validation does not imply that every assertion inside the artifact is true.

### 5.2 Assertion-level validation

Assertion-level validation answers:

* Is this particular assertion supported?
* By which evidence?
* Under which authority channel?
* Under which validation policy?
* Is it disputed, rejected, hypothesis-grade, reviewed, or validated?
* Does it conflict with assertions in other shards or authority channels?

### 5.3 Shard-level validation

Shard-level validation answers:

* Is this scoped shard accepted by a declared authority channel?
* Does it satisfy the policy for its scope?
* Are required validators, attestations, or references present?
* Is it eligible for inclusion in a given federation?

### 5.4 Federation-level validation

Federation-level validation answers:

* Which shards are included?
* Which authority registry is applied?
* Which conflict policy is used?
* Which shards are mandatory or optional?
* Which authority channels are preferred for overlapping scopes?
* Which claims remain disputed or rejected across channels?

A federation may be valid even when some included claims are disputed, provided the dispute is explicit and the federation policy allows that state.

---

## 6. Plural authority model

Kristal v5 does not define one universal truth authority.

Different authorities may approve different versions of the same corpus, shard, or assertion. A government, association, company, research collective, religious community, standards body, or international organization may publish or recognize its own Kristal.

This is not a defect. It is a core feature of the model.

The system MUST therefore prevent authority laundering:

> A claim validated by one authority channel MUST NOT be presented as validated by another authority channel unless that second channel explicitly recognizes it.

For example:

* A mythology corpus may be validated as mythology, literature, culture, or religious tradition.
* The same mythology corpus MUST NOT be represented as validated physical-world fact unless an authority channel explicitly validates it under that epistemic mode.
* A fringe scientific corpus may exist and may be internally recognized by a community, but it MUST remain marked as rejected, disputed, or unrecognized by scientific authority channels that reject it.
* A company may be the primary authority for describing its own internal systems, but other authorities may still be required for safety, security, environmental, legal, or interoperability claims.

---

## 7. Wikidata Seed Kristal

The first global Kristal v5 corpus MAY be a **Wikidata Seed Kristal**.

This should not be described as a semantic transformation of Wikidata into another truth system. The v1 seed is a Kristal-compatible representation, packaging, and alignment of the Wikidata corpus.

The Wikidata Seed Kristal SHOULD preserve, as far as possible:

* Wikidata identifiers;
* statement structure;
* references;
* qualifiers;
* ranks;
* source metadata;
* update lineage;
* export compatibility;
* query compatibility.

Kristal-specific additions MAY include:

* artifact identity;
* stronger reproducibility metadata;
* explicit authority-channel recognition;
* validation and review bundles;
* federation metadata;
* Runtime Pack metadata;
* certainty and epistemic-mode annotations where not already represented.

A Wikidata Seed Kristal being recognized as a global reference corpus does not mean that every statement in the corpus is universally true. It means that the corpus identity, packaging, provenance, compatibility, and role as a global seed reference are recognized.

---

## 8. UNESCO and global reference recognition

UNESCO, or a similar global institution, MAY act as a high-level reference authority channel or recognizer for selected global Kristal federations.

Such recognition SHOULD mean one or more of the following:

* recognition of a seed corpus as a global reference corpus;
* recognition of an authority channel for a domain;
* recognition of a federation policy;
* recognition of validation processes used by domain authorities;
* recognition of selected reference shards or corpora.

UNESCO recognition SHOULD NOT mean that UNESCO manually validates every assertion.

For example:

* If the World Health Organization submits or recognizes a medical Kristal, UNESCO may recognize WHO as the relevant authority channel for the medical scope.
* If a company publishes a Kristal describing its own systems, the company may be authoritative for internal system description, while safety, privacy, environmental, and interoperability claims may require other authorities.
* If an inventor submits a Kristal for a working anti-pollution machine, recognition may depend on prototypes, measurements, laboratories, environmental authorities, reproducibility, and domain-specific review.

UNESCO does not own Kristal truth. It may recognize reference channels, corpora, shards, and validation policies within declared scopes.

---

## 9. Divergent forks and creative corpora

Kristal v5 MUST permit divergent forks, creative corpora, research forks, speculative corpora, and alternative world-models.

A fork MUST preserve:

* source lineage;
* source artifact identity;
* fork publisher identity;
* changed assertions;
* authority channel;
* scope;
* validation policy;
* epistemic mode;
* recognition status.

A fork MAY be valid inside one authority channel while being rejected by another.

Reader systems SHOULD expose this clearly. For example:

```json
{
  "assertion_id": "sha256:...",
  "claim": "example claim",
  "recognized_by": ["authority:local-community"],
  "rejected_by": ["authority:scientific-reference-channel"],
  "epistemic_mode": "disputed",
  "default_visibility": "alternate-view",
  "federation_conflict_status": "conflicts-with-reference-channel"
}
```

Default public views SHOULD prioritize the authority channels configured for that deployment. Research or exploration modes MAY expose broader forks, including rejected or fringe corpora, provided the status and authority separation remain clear.

---

## 10. Role of AI validation

AI systems MAY assist validation by producing:

* consistency checks;
* evidence retrieval;
* contradiction detection;
* schema validation support;
* provenance review;
* comparison against existing reference corpora;
* suggested confidence levels;
* review summaries;
* reproducibility checks.

AI output MUST be treated as a validation artifact or review signal unless an authority channel explicitly gives that AI validator a defined authority role.

AI validation MUST remain attributable, reproducible where declared, scoped, and policy-bound.

---

## 11. Reader and ecosystem behavior

Reader systems, query systems, creative tools, research tools, and kOA ecosystem tools MAY read, duplicate, fork, extend, annotate, and repackage Kristals.

They MUST NOT silently collapse authority channels or validation scopes.

A reader SHOULD be able to answer:

* Who published this?
* Who recognizes it?
* Under which scope?
* Under which epistemic mode?
* Which assertions are disputed?
* Which authority channels reject it?
* Which federation policy produced this view?
* Is this a seed corpus, a fork, a reference view, a fictional corpus, a working artifact, or a validated domain shard?

Konnaxion, Orgo, Architect, and SenTient may interact with Kristals differently:

* Konnaxion distributes, searches, activates, and presents packs or federated views.
* Orgo manages workflows, review, operational escalation, approvals, and lifecycle.
* Architect renders human-readable outputs from declared query results without inventing facts.
* SenTient resolves ambiguity and preserves unresolved states rather than forcing interpretation.

---

## 12. Compatibility with v4

This document preserves the following v4 invariants:

* deterministic builds;
* content-addressed identity;
* canonicalization and hashing rules;
* signature and trust-root verification;
* offline verification;
* Runtime Pack reproducibility;
* explicit provenance;
* preservation of ambiguity;
* deterministic federation composition;
* authority registry and revocation policy checks.

This document revises the following v4 assumptions:

* `Claim-IR` is no longer the only universal proposal boundary.
* Validation is no longer a universal compilation gate.
* “Canonical” should not be used as unqualified universal truth.
* Compilation does not imply truth recognition.
* A Runtime Pack may be derived from a working, disputed, fictional, mythological, research, or recognized reference corpus, provided its trust metadata makes that status explicit.
* Federation is not merely package composition; it is also the mechanism by which plural authority and divergent corpora coexist without being merged silently.

---

## 13. Normative summary

Kristal v5 is defined by these rules:

* A Kristal MAY exist before expert validation.
* A Kristal MAY contain errors, hypotheses, disputes, fiction, mythology, and speculative claims.
* Artifact validity MUST be distinguished from assertion truth.
* Recognition MUST be scoped by authority channel, policy, and domain.
* No authority channel has an automatic global monopoly.
* Federation MUST preserve source identity, authority identity, lineage, and conflict visibility.
* A claim recognized by one authority MUST NOT be silently treated as recognized by another.
* A seed corpus MAY be globally recognized without implying that every internal assertion is universally true.
* AI MAY assist validation but does not become authority unless a policy explicitly grants that role.
* Reader systems MUST preserve epistemic status and authority separation.

---

## 14. One-sentence definition

**Kristal v5 is a deterministic, federated epistemic artifact system that lets uncertain, creative, disputed, and validated knowledge coexist while preserving scope, authority, provenance, and recognition boundaries.**
