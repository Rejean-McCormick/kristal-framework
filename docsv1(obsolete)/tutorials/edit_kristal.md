
# Edit a Kristal (Offline Improvement Workflow)

## Status
DRAFT

## 1. Goal

This tutorial explains how a user edits a Kristal offline while preserving:
- Q/P/R/S alignment,
- deterministic claim identity,
- traceability (without making the Kristal heavy),
- mergeability back into upstream Kristals.

A Kristal edit typically means:
- adding new claims,
- correcting claims,
- adding missing Q-items (entities),
- refining properties (R/S usage),
- attaching evidence references and trace refs.

---

## 2. Recommended Workflow

### Step 1 — Create a working copy
Never edit the original downloaded Kristal directly. Create a local working copy:

```bash
cp plants.core.kristal.json plants.core.local.kristal.json
````

### Step 2 — Decide your edit scope

Common scopes:

* **Entity-focused**: improve one Q-item neighborhood (e.g., `Q123`)
* **Domain-focused**: fill gaps for a domain pack (e.g., missing properties/constraints)
* **Evidence-focused**: upgrade inferred claims to asserted claims by adding sources

---

## 3. What You Can Edit Safely

### 3.1 Add a new claim

Add a new claim object to `claims[]`:

```json
{
  "claim_id": "claim://sha256:<computed>",
  "subject": "Q123",
  "property": "P279",
  "value": { "type": "wikibase-entityid", "content": "Q999" },
  "qualifiers": [],
  "references": [
    {
      "source_ref": "source://sha256:<hash>",
      "locator": { "url": "https://example.org", "offset": [120, 220] }
    }
  ],
  "confidence": 0.90,
  "status": "asserted",
  "trace_refs": ["trace://sha256:<trace_root>"]
}
```

Rules:

* `subject` must be a QID present in `qitems[]` (or added by you).
* `property` must be defined in available charters (P/R/S).
* `value` must match the charter value type.
* Avoid embedding full evidence text; use `source_ref` + locator.

### 3.2 Add a new entity (Q-item)

If you introduce a new QID or a local placeholder, add it in `qitems[]`:

```json
{
  "id": "Q_tmp_a1b2c3",
  "labels": { "en": "Local Species X" },
  "descriptions": { "en": "Placeholder entity created offline." },
  "aliases": {}
}
```

If you later resolve it to a real `Q####`, you must update all claims referencing it.

### 3.3 Refine a predicate using R/S

If your meaning is workflow-specific or Orgo-specific:

* prefer `R####NNN` when it refines a known Wikidata property `P####`
* use `S####NNN` (or `S0000NNN`) when no safe Wikidata mapping exists

**Do not create new R/S IDs locally unless you also update the charter.**

---

## 4. Edits That Require Charter Changes

You must update charters if you:

* add a new `R####NNN` refinement,
* add a new `S####NNN` property,
* change a property’s `value_type` or constraints.

Typical workflow:

1. edit charter JSON (embedded or referenced local file)
2. re-run validation
3. recompile claim IDs if needed (because constraints affect validity)

---

## 5. Claim Identity and IDs

Kristals rely on stable `claim_id`s to support merging.

### 5.1 When you MUST recompute claim_id

Recompute the `claim_id` if you change any of:

* subject
* property
* value
* qualifiers
* reference hashes/refs (if your hashing includes them)

### 5.2 When you should NOT change claim_id

Do NOT change `claim_id` just because:

* confidence changed
* status changed
* trace_refs changed

(These should not affect claim identity.)

---

## 6. Trace Logging for Edits

Edits should be traceable without bloating the Kristal.

Recommended:

* create a local `trace.bundle.json` for your edit session
* include:

  * who edited
  * what was changed
  * evidence hashes
  * reasoning notes if needed

Then reference it from:

* claim `trace_refs[]`
* and optionally `manifest.trace_root`

---

## 7. Validation Before Saving

After edits, run validation:

```bash
kristal validate --in plants.core.local.kristal.json
```

Validation should check:

* schema correctness
* missing Q-items
* unknown P/R/S properties
* value type mismatches
* missing required evidence (policy-defined)
* duplicate/conflicting claims

If validation fails:

* either fix issues, or
* mark claims as `needs_review` and lower confidence.

---

## 8. Suggested Local Layout

```
./work/
  plants.core.local.kristal.json
  charters/
    general.charter.json
    plants.domain.charter.json
  traces/
    trace.bundle.local.json
  reports/
    validation.report.json
```

---

## 9. Next Steps

* Reload the Kristal into your local engine/AI runtime: `docs/en/tutorials/reload_kristal.md`
* Propose your changes for merging: `docs/en/contributors/propose_change.md`


