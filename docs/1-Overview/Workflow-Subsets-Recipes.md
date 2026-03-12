# Workflow: Subsets (Recipes)

Subsets let you build **smaller, focused Kristals** from a larger source snapshot, while keeping the build **deterministic and auditable**.

Typical reasons to create subsets:
- A single domain (e.g., heritage, medicine, finance)
- A language slice (e.g., en/fr only)
- A time window (e.g., last 12 months)
- A tenant/project-specific view
- A “starter” package for faster distribution and testing

## What you produce
Depending on your pipeline, a subset build typically produces:

- **Exchange** (the subset Kristal)
- **(Optional) Runtime Pack** (optimized files/indexes for serving/query)
- **Validation Report(s)** (evidence the subset meets required profiles)
- **(Optional) Subset Recipe artifact** (the explicit recipe used to generate the subset)

## High-level steps
### 1) Choose the source snapshot
Start from a **pinned input snapshot** (the thing you’re slicing). The snapshot must be stable and content-addressed (or otherwise versioned) so the subset can be reproduced later.

### 2) Define the recipe (the “what to include”)
A recipe usually answers:
- **Seeds**: starting entities / entry points
- **Expansion rules**: how to traverse links/claims from the seeds
- **Constraints**: allow/deny lists, language filters, domain filters
- **Stopping criteria**: max depth, max entities, max statements, etc.

Keep recipes **declarative**: avoid “implementation magic” that can’t be reproduced.

### 3) Build deterministically
Run the build with a fixed configuration:
- deterministic ordering policies
- pinned compiler/config hash
- pinned profile versions

The goal: **same inputs + same recipe + same policies ⇒ same output**.

### 4) Validate and generate evidence
Run the required validation profiles (your org’s gating rules). Produce Validation Reports and attach references as appropriate.

### 5) Publish / distribute / activate
Publish the subset as you would any Kristal artifact:
- distribute the Exchange (and optional Runtime Pack)
- activate in the target environment
- monitor query behavior and performance

## What “done” looks like
- The subset’s Exchange is **content-addressed** and reproducible
- The subset is **validated** under the required profiles
- The subset can be **re-built** later and yield identical IDs/hashes (within declared determinism rules)
- Consumers can verify **integrity/trust** using declared signatures/policies

## Common pitfalls
- **Unpinned inputs**: “latest dump” without a stable snapshot reference
- **Hidden config drift**: compiler config changes without a new config hash
- **Non-deterministic traversal**: relying on unordered iteration
- **Recipe ambiguity**: rules that depend on runtime environment or external services

## Tech details
- Subset recipes (spec/guidance): `kristal-docs-v4/03-reproducibility/subset-recipes.md`
- Deterministic build rules: `kristal-docs-v4/03-reproducibility/deterministic-build-rules.md`
- Exchange schema: `kristal-docs-v4/02-schemas/exchange-manifest.schema.json`