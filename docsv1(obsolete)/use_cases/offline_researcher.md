
# Use Case: Offline Researcher (Fieldwork / No Network)

## Status
DRAFT

## 1. Scenario

An offline researcher works in environments where network access is unreliable, restricted, or unavailable:
- fieldwork (remote areas),
- secure facilities (air-gapped labs),
- travel with limited connectivity,
- archives or sites with device restrictions.

They need:
- a **bounded** knowledge base relevant to their domain,
- local querying and slicing,
- the ability to annotate and improve knowledge offline,
- eventual synchronization/merge when online again.

Kristals enable this by packaging knowledge into portable, versioned artifacts.

---

## 2. Goals

- **Offline-first usability**: no dependency on a central database
- **Fast local retrieval**: query like a database on-device
- **Deterministic updates**: stable IDs and diff-friendly ordering
- **Local improvements**: add claims, correct errors, attach evidence
- **Safe provenance**: keep evidence as hashes/locators; optionally store excerpts
- **Sync later**: merge changes into upstream packs or registries

---

## 3. Typical Workflow

### 3.1 Before going offline: download domain packs
The researcher downloads one or more Kristals:

```bash
kristal pull kristal://<domain>.core --out ./kristals/
````

Optional:

* include indexes for fast resolution
* include trace bundles if audit/explanations are needed offline

```bash
kristal pull kristal://<domain>.core --with-index --traces refs
```

### 3.2 Create a working copy

```bash
cp <domain>.core.kristal.json <domain>.local.kristal.json
```

### 3.3 Query locally during research

Local queries retrieve structured facts and relationships:

```sparql
SELECT ?p ?o WHERE { Q<seed> ?p ?o . }
```

### 3.4 Record new knowledge offline

Add:

* new claims (`claims[]`)
* new Q-items (`qitems[]`) if necessary
* references (hashes + locators)
* optional trace bundle entries (notes, extraction history)

### 3.5 Validate locally

```bash
kristal validate --in <domain>.local.kristal.json
kristal test --in <domain>.local.kristal.json
```

### 3.6 When online again: merge/sync

The researcher proposes changes:

* Kristal diff
* validation report
* optional trace bundle

See contributor workflow:

* `docs/en/contributors/propose_change.md`

---

## 4. What Makes Offline Packs Work

### 4.1 Bounded scope

Offline packs must be scoped:

* by domain tags (e.g., UNESCO codes),
* by topic seed (QID neighborhood),
* by query-built slices.

### 4.2 Closure completeness

When building an offline pack, slicing must include:

* all referenced QIDs (objects),
* optionally ontology parents (type chains) needed for reasoning,
* required property catalogs (charters).

### 4.3 Local indexes

For usability, packs should include:

* label/alias → QID lookup
* property label → P/R/S lookup
* optional similarity indexes (embeddings) if hardware permits

---

## 5. Offline Evidence Handling

Evidence may be large or sensitive. Recommended:

* store **hashes + locators** in the Kristal core
* store excerpts only in optional trace bundles

Example reference stub:

```json
{
  "source_ref": "source://sha256:<hash>",
  "locator": { "url": "file://local.pdf", "page": 12 }
}
```

If the researcher cannot keep the document:

* keep only the hash + locator metadata for later verification.

---

## 6. Offline AI Usage

An AI runtime can use Kristals offline by:

1. mapping user text to candidate QIDs using the local index,
2. retrieving a relevant subgraph via local queries,
3. generating answers grounded in claims.

Recommended defaults:

* use only high-confidence asserted claims
* include inferred/contested claims only on explicit request

---

## 7. Risks and Mitigations

### Risk: unresolved entities while offline

Mitigation:

* allow placeholders (`Q_tmp_*`)
* mark claims `needs_review`
* keep trace notes for later resolution

### Risk: property mismatch (unknown P/R/S)

Mitigation:

* ensure charters are bundled
* disallow creating new R/S properties offline unless charter edits are included

### Risk: knowledge drift

Mitigation:

* keep lineage metadata
* rebase local changes on latest upstream when reconnecting
* run merge and conflict checks

---

## 8. Why Kristals Help Offline Researchers

* no central dependency
* database-like local queries
* deterministic mergeable updates
* provenance without bloating files
* AI grounding without internet


