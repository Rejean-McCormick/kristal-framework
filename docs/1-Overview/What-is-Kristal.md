# What is Kristal?

Kristal is a framework for producing and operating **deterministic knowledge artifacts** (“Kristals”) that can be **validated, distributed, and queried** reliably—even when offline.

A Kristal is not “a database”. It is a **portable artifact** with:
- a stable identity,
- explicit inputs and build metadata,
- integrity and (optionally) trust signals,
- and predictable query behavior once activated.

---

## Why Kristal exists

Most knowledge systems fail in at least one of these ways:
- results depend on hidden runtime state (non-deterministic)
- you cannot reproduce the same artifact from the same inputs
- you cannot verify what you received (integrity/trust gaps)
- you cannot safely roll forward/backward across versions

Kristal is designed to make these properties **first-class**:
- deterministic builds (reproducibility)
- content-addressed identity
- fail-closed verification when declared
- operationally safe distribution (activation/rollback)

---

## What a Kristal enables (high level)

- **Build** a Kristal from upstream sources
- **Validate** it with schemas/profiles
- **Publish** it as a versioned artifact
- **Activate** it into a runtime environment (locally or in services)
- **Query** it with declared capabilities
- **Evolve** it safely (patches, rollbacks, compatibility rules)

---

## Core building blocks

### Artifacts
Kristal workflows revolve around a small set of artifacts you can store, move, and verify:
- Exchange
- Runtime Pack
- Validation Report
- (Optional) Shard Manifest, Federation Manifest, Authority Registry

See: [Artifacts](Artifacts)

### Workflows
Workflows describe how teams operationalize Kristal:
- Build & Validate
- Publish & Distribute
- Activate, Rollback & Downgrade
- Subsets (Recipes)
- Federation & Curation

See: [Workflows](Workflows)

### Concepts
These concepts show up everywhere:
- **Identity & determinism:** what “same inputs → same artifact” means in practice  
  See: [Identity & Determinism](Identity-and-Determinism)
- **Trust & authority:** who can sign, how trust roots are used, and what “fail-closed” means  
  See: [Trust, Authority & Signatures](Trust-Authority-and-Signatures)

---

## Where to go next

- If you want an end-to-end path: [Quickstart](Quickstart)
- If you want the mental model: [Concepts & Mental Model](Concepts-and-Mental-Model)
- If you want to understand the artifacts: [Artifacts](Artifacts)
- If you want to operate Kristal in production: [Operations](Operations)

---

## Technical details

This wiki intentionally avoids repeating full specifications. For exact schemas and normative rules, refer to the technical docs in the main repository.