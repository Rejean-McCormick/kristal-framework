# Subset recipes (Kristal v3 reproducibility)

## Status

Draft (normative for subset builder reproducibility)

## Purpose

A **subset recipe** is a declarative, reproducible input that defines how to derive a **deterministic subset** of a larger knowledge source snapshot so that the resulting artifacts (Exchange and Runtime Pack) are reproducible and comparable across toolchains. Subset recipes are first-class in v3: they specify seeds, deterministic expansion rules, allow/deny constraints, depth limits, stopping criteria, and snapshot identifiers. :contentReference[oaicite:0]{index=0} :contentReference[oaicite:1]{index=1}

Normative keywords: MUST, MUST NOT, SHOULD, SHOULD NOT, MAY.

---

## 1. Scope and non-goals

### 1.1 In scope

- Recipe object model (fields and semantics)
- Deterministic subset selection and expansion rules
- Deterministic truncation / stopping criteria
- Recipe identity (content-addressed `recipe_id`)
- How to record recipe references in build manifests

### 1.2 Out of scope

- Full Wikidata subset research algorithms (this spec provides a constrained, portable baseline)
- Query/runtime behavior (see `04-query/query-contract.md`)
- Runtime Pack layout policies (see `03-reproducibility/allowed-runtime-pack-policies.md`)

---

## 2. Determinism contract (core)

### 2.1 Deterministic subset requirement

Given identical:
- **source snapshot** (content-addressed identifier),
- **subset recipe** (content-addressed identifier),
- **compiler identity/version**,
- and any declared **policies/profiles** affecting selection,

then the subset builder MUST produce identical **selected sets**:
- selected entities,
- selected statements/claims,
- and any selected evidence blobs (if included),

and MUST therefore be able to reproduce identical downstream artifacts. Subset-recipe inputs are explicitly part of the build input snapshot. :contentReference[oaicite:2]{index=2}

### 2.2 Offline constraint

Subset evaluation MUST be executable without network access. Any external dependency MUST be represented as a content-addressed input snapshot referenced by the recipe (or by the build manifests).

---

## 3. Recipe identity (content-addressed)

### 3.1 `recipe_id`

A subset recipe MUST have a stable content-addressed identifier:

- `recipe_id = "sha256:" + hex(SHA-256(JCS(hash_target(recipe))))`

Where:
- `JCS` is RFC 8785 JSON Canonicalization Scheme (same canonicalization family used by v3 core artifacts).
- `hash_target(recipe)` is the recipe object with:
  - `recipe_id` removed (output field),
  - and any signature/attestation envelope removed if present.

### 3.2 Canonical recipe representation

The normative recipe format is a **JSON object**. If a producer stores recipes in another syntax (YAML, TOML, etc.), it MUST define a deterministic mapping into the normative JSON object prior to hashing, and the resulting JSON object MUST be the one used to compute `recipe_id`.

---

## 4. Recipe object model (normative)

A conforming recipe MUST be a JSON object with the following top-level fields.

### 4.1 Required fields

- `spec_version` (string)
  - MUST be `"subset-recipe@1"` for this version of the spec.
- `recipe_id` (string)
  - MUST be present and MUST match the computed content-addressed ID (Section 3.1).
- `source_snapshot_id` (string)
  - Content-addressed identifier of the upstream dataset/snapshot this recipe is evaluated against.
- `seeds` (object)
  - Seed sets that anchor the subset.
- `expansion` (object)
  - Deterministic expansion rules.
- `constraints` (object)
  - Allow/deny lists and other guards.
- `stopping` (object)
  - Depth/budget/termination rules.

### 4.2 Optional fields

- `recipe_version` (string)
  - Human-managed semantic version for the recipe’s intent (does not replace `recipe_id`).
- `description` (string)
- `profiles_enabled` (array of strings)
  - Explicit optional profiles that affect selection behavior (if any).
- `signatures` (array)
  - Optional; MUST NOT affect `recipe_id` (Section 3.1).

---

## 5. Seeds (normative)

### 5.1 Seed kinds

`seeds` MUST include at least one of:

- `entity_ids`: array of entity identifiers (e.g., QID-like strings)
- `property_ids`: array of property identifiers (e.g., PID-like strings)
- `statement_ids`: array of stable statement identifiers (if your source snapshot exposes them)

### 5.2 Deterministic seed normalization

Before evaluation:
- Seed arrays MUST be deduplicated.
- Seed arrays MUST be sorted deterministically using a stable byte-wise lexicographic order over their canonical string form.
- Empty/unknown identifiers MUST be rejected (validation error).

---

## 6. Expansion rules (normative baseline)

`expansion.rules` MUST be an array of rule objects. Each rule object MUST have:
- `rule_id` (string, non-empty; unique within the recipe)
- `kind` (string enum; see below)
- `params` (object; parameters for the rule)

Rules are applied in a deterministic pipeline:
1. Seeds initialize the frontier at depth 0.
2. For each depth `d` from 0..`max_depth` (inclusive), apply rules in recipe order.
3. All sets produced at each step MUST be merged with deterministic deduplication and deterministic ordering (see Section 8).

### 6.1 Allowed rule kinds (portable baseline)

Implementations claiming conformance to `subset-recipe@1` MUST implement at least:

1) `OUTGOING_EDGES`
- Meaning: for each entity in frontier, include statements where the entity is the subject, and optionally add referenced target entities to the next frontier.
- Required params:
  - `predicates`: array of property IDs, or `"*"` to mean “all predicates allowed by constraints”
  - `include_targets`: boolean (if true, entity-valued objects become candidates for next frontier)
  - `include_qualifiers`: boolean
  - `include_references`: boolean

2) `INCOMING_EDGES`
- Meaning: for each entity in frontier, include statements where the entity appears as an entity-valued object, and optionally add the statement subjects to the next frontier.
- Required params:
  - `predicates`: array of property IDs, or `"*"`
  - `include_subjects`: boolean (if true, statement subjects become candidates for next frontier)
  - `include_qualifiers`: boolean
  - `include_references`: boolean

3) `TAXONOMY_CLOSURE`
- Meaning: include closure over a declared taxonomy predicate set (e.g., subclass-of / instance-of equivalents) up to a bound.
- Required params:
  - `predicates`: array of property IDs (taxonomy edges)
  - `direction`: `"up" | "down" | "both"`
  - `max_hops`: integer >= 1

4) `LABELS_AND_DESCRIPTIONS`
- Meaning: include label/alias/description facts required for UX.
- Required params:
  - `languages`: array of BCP-47 tags or `"*"`
  - `include_aliases`: boolean
  - `include_descriptions`: boolean

If an implementation supports additional rule kinds, they MUST be declared as a profile (via `profiles_enabled`) and MUST remain deterministic and testable.

---

## 7. Constraints (allow/deny) (normative)

`constraints` MUST include:

- `allow_entities` (array) and `deny_entities` (array)
- `allow_properties` (array) and `deny_properties` (array)

Rules:
- Deny lists MUST take precedence over allow lists.
- If an allow list is non-empty, only identifiers present in it are eligible (after applying deny precedence).
- Constraints MUST apply consistently to:
  - seed sets,
  - expansion results (statements and next-frontier candidates),
  - and truncation tie-breaking sets.

---

## 8. Deterministic ordering and truncation (mandatory)

### 8.1 Stable ordering keys

All intermediate and final sets MUST be ordered deterministically:

- **Entity ordering key**: canonical string ID (byte-wise lexicographic).
- **Property ordering key**: canonical string ID (byte-wise lexicographic).
- **Statement ordering key** (preferred):
  1) stable `statement_id` if available, else
  2) deterministic surrogate key computed as JCS+SHA256 over a canonical tuple of `(subject_id, predicate_id, object_value, qualifiers, references)` with stable ordering inside nested structures.

### 8.2 Budget limits and deterministic truncation

When a stopping/budget limit is reached (Section 9), the subset builder MUST truncate deterministically:

- It MUST sort candidates by the relevant stable ordering key.
- It MUST select the first `N` items deterministically.
- It MUST record (in build metadata / manifest extensions) that truncation occurred, including which limit triggered it and the resulting counts.

Silent, non-declared truncation MUST NOT occur.

---

## 9. Stopping criteria (normative)

`stopping` MUST include:

- `max_depth` (integer >= 0)
- and at least one budget:
  - `max_entities` (integer >= 1) OR
  - `max_statements` (integer >= 1)

Optional:
- `max_evidence_blobs` (integer)
- `max_frontier_per_depth` (integer; must truncate deterministically if exceeded)

Stopping rules:
- Depth is evaluated as BFS depth from seeds.
- Budgets are global across the run unless a profile explicitly defines per-depth budgets.

---

## 10. Recording subset recipes in manifests (normative guidance)

Subset recipes participate in determinism as inputs. :contentReference[oaicite:3]{index=3}

### 10.1 Exchange Manifest

If a subset recipe was used to produce an Exchange:
- The Exchange Manifest MUST record the recipe reference in a deterministic way.
- If the manifest schema does not yet provide a dedicated `inputs.subset_recipe` field, producers SHOULD record:
  - `extensions.subset_recipe.recipe_id`
  - `extensions.subset_recipe.spec_version`
  - `extensions.subset_recipe.source_snapshot_id`

The Exchange manifest schema explicitly allows `extensions` for implementation-specific fields that must not change core identity/hashing semantics. :contentReference[oaicite:4]{index=4}

### 10.2 Runtime Pack Manifest

If a Runtime Pack was compiled from an Exchange built via a subset recipe:
- The Runtime Pack Manifest SHOULD also record the subset recipe reference (directly, or by carrying the Exchange Manifest hash + a deterministic pointer in extensions).

---

## 11. Conformance tests (recommended)

Implementations SHOULD add subset recipe fixtures to the reproducibility test corpus:

- **SR-1 (same toolchain)**: same `source_snapshot_id` + same recipe JSON → identical selected entity/statement sets
- **SR-2 (cross toolchain)**: published recipe fixture with expected `recipe_id` and expected selected set hashes → independent implementations match
- **SR-3 (budget truncation)**: force truncation and verify deterministic truncation + explicit truncation declaration

---

## 12. Minimal example (non-normative)

```json
{
  "spec_version": "subset-recipe@1",
  "recipe_id": "sha256:<computed>",
  "source_snapshot_id": "sha256:<snapshot>",
  "seeds": {
    "entity_ids": ["Q42", "Q1"],
    "property_ids": ["P31"]
  },
  "expansion": {
    "rules": [
      {
        "rule_id": "r1",
        "kind": "OUTGOING_EDGES",
        "params": {
          "predicates": "*",
          "include_targets": true,
          "include_qualifiers": true,
          "include_references": true
        }
      },
      {
        "rule_id": "r2",
        "kind": "LABELS_AND_DESCRIPTIONS",
        "params": {
          "languages": ["en", "fr"],
          "include_aliases": true,
          "include_descriptions": true
        }
      }
    ]
  },
  "constraints": {
    "allow_entities": [],
    "deny_entities": [],
    "allow_properties": [],
    "deny_properties": []
  },
  "stopping": {
    "max_depth": 2,
    "max_entities": 50000,
    "max_statements": 200000
  }
}