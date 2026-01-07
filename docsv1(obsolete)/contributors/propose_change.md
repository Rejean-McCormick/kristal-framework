
# Propose a Change (Contribution Workflow)

## Status
DRAFT

## 1. Goal

This guide explains how contributors propose changes to Kristals while preserving:
- deterministic claim identity,
- Q/P/R/S alignment (Orgo charters),
- auditability via detached traces,
- mergeability and conflict review.

A “change” can be:
- adding or correcting claims,
- adding missing Q-items (entities),
- adding evidence references,
- proposing new R/S properties (requires charter update),
- improving validation rules or indexes.

---

## 2. Recommended Contribution Pattern

### 2.1 Fork / Branch
Work on a dedicated branch:

```bash
git checkout -b kristal/<domain>/<short-change-name>
````

### 2.2 Keep changes atomic

Prefer small, reviewable changes:

* one domain pack at a time,
* one charter change at a time,
* one focused set of entities.

---

## 3. What to Include in a Change Proposal

A proposal should include:

1. **Kristal core diff**

* changed `claims[]` and/or `qitems[]`
* updated manifest (lineage, created_at optional for derived packs)

2. **Trace bundle(s)** (optional but recommended)

* proofs/evidence hashes or locators
* decision logs (entity/property selection)
* validation report

3. **Validation report**

* output from `kristal validate`
* list of warnings accepted (if any)

4. **Charter updates** (only if needed)

* adding/modifying `R####NNN` or `S####NNN` definitions
* updates to constraints / value types

---

## 4. How to Prepare Your Patch

### Step 1 — Edit the Kristal

Follow the editing tutorial:

* `docs/en/tutorials/edit_kristal.md`

### Step 2 — Recompute IDs (only when needed)

You MUST recompute `claim_id` if you changed:

* subject
* property
* value
* qualifiers
* reference hashes (if included in claim identity)

You should NOT recompute `claim_id` only for:

* confidence updates
* status updates
* trace_refs changes

### Step 3 — Validate

Run validation and save the report:

```bash
kristal validate --in <kristal-file> --out reports/validation.report.json
```

### Step 4 — Add traces (recommended)

If you have evidence excerpts, resolution decisions, or model outputs:

* store them in a detached trace bundle
* reference them from claim `trace_refs[]`

Example layout:

```
traces/
  trace.bundle.<change>.json
reports/
  validation.report.json
```

---

## 5. Special Case: Proposing New R/S Properties

If your change requires a new property:

* create/update the relevant charter JSON
* assign an ID following the Orgo rules:

  * R####NNN refines P####
  * S####NNN based_on P####
  * S0000NNN when based_on is null

Then:

* use the new property ID in claims
* run validation again (constraints now apply)

A property proposal should include:

* `id`
* `based_on` (or null)
* `label_en`
* `description_en`
* `value_type`
* minimal constraints (cardinality, allowed values if needed)

---

## 6. Proposed Change Format (PR / Patch Template)

Include a short markdown file in your PR (or commit message) with:

```markdown
## Change Summary
- Domain: <domain pack or kristal name>
- Scope: <entities/claims>
- Motivation: <why>

## What Changed
- Added claims: <count>
- Modified claims: <count>
- Removed claims: <count>
- Added qitems: <count>
- Charter changes: <yes/no>

## Evidence & Traces
- Trace bundle: traces/trace.bundle.<name>.json
- Evidence sources: <list of URLs or source hashes>

## Validation
- Report: reports/validation.report.json
- Status: passed/partial
- Warnings accepted: <list>
```

---

## 7. Review Expectations

Reviewers typically check:

* property correctness (P/R/S) against charters
* entity correctness (Q resolution)
* value type correctness
* evidence presence for asserted claims (policy-defined)
* claim duplication/conflicts
* stability of IDs and ordering

---

## 8. Merge Outcomes

A proposal may be merged as:

* **accepted into core** (claims promoted to asserted/core context)
* **accepted but marked inferred** (kept, but lower trust)
* **kept as contested** (parallel claims)
* **rejected** (kept only in traces or dropped)

---

## 9. Next Steps

* Test your Kristal locally: `docs/en/contributors/test_kristal.md`
* Validate versioning and IDs: `docs/en/contributors/validate_version.md`


