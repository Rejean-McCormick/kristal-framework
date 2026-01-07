
# Use Case: Botanist (Domain Kristal for Plants)

## Status
DRAFT

## 1. Scenario

A botanist wants a high-signal, offline-capable knowledge base for plant science:
- taxonomy (species, genus, family),
- morphology (leaf, flower, fruit features),
- habitats and geographic distributions,
- interactions (pollinators, diseases),
- citations and evidence references.

They do not want the entire global knowledge graph—only what is relevant to plants.

Kristals enable this by providing a **domain pack** (“Plants”) that can be:
- downloaded,
- queried locally,
- edited and enriched by the botanist,
- merged back into an upstream Kristal ecosystem.

---

## 2. Goals

- **Offline research**: fieldwork or remote areas without internet
- **Fast local queries**: taxonomy lookups, property constraints, relationship traversals
- **Expert corrections**: add missing species, correct classifications, add evidence
- **Controlled provenance**: evidence references and trace logs when needed
- **Mergeable contributions**: submit improvements without breaking alignment

---

## 3. Downloading the Plant Kristal

### 3.1 Curated domain pack
The botanist downloads a curated pack:

```bash
kristal pull kristal://plants.core --out ./kristals/
````

The pack includes:

* plant-relevant Q-items
* required P/R/S properties (charters)
* plant-focused claims
* optional indexes for faster entity resolution

### 3.2 UNESCO-aligned discovery (optional)

If the registry supports domain tags:

```bash
kristal search --domain unesco:<plant-code>
kristal pull --domain unesco:<plant-code> --best
```

---

## 4. Local Query Examples

### 4.1 Get the taxonomy neighborhood for a species

Retrieve classification edges:

```sparql
SELECT ?p ?o WHERE { Q<species> ?p ?o . }
```

Typical outputs:

* `P31` instance of (species)
* `P171` parent taxon (genus/family)
* `P279` subclass of (if used in the pack)

### 4.2 List all entities that are plants (or subclasses)

```sparql
SELECT ?s WHERE { ?s P31 Q<plant> . }
```

### 4.3 Find species in a geographic region

```sparql
SELECT ?s WHERE { ?s P131 Q<region> . }
```

(Exact properties may be P/R/S depending on the charter.)

---

## 5. Editing: Expert Improvements

### 5.1 Add a missing species

The botanist adds:

* a new Q-item (if not already present)
* claims describing taxonomy and traits
* evidence references to authoritative sources

New claims are added with:

* correct QIDs (or placeholders marked `needs_review`)
* correct properties (P/R/S)
* typed values
* confidence and status

### 5.2 Correct a classification error

If the botanist finds:

* wrong parent taxon
* outdated taxonomy

They can:

* add a corrected claim in `ctx:local`
* mark the old one as contested (policy-dependent)
* attach evidence references

### 5.3 Add evidence without bloating the Kristal

Instead of embedding long excerpts, the botanist:

* stores evidence hashes and locators in `references[]`
* optionally stores excerpts in a detached trace bundle
* links it using `trace_refs[]`

---

## 6. Validation and Testing

Before sharing:

```bash
kristal validate --in plants.core.local.kristal.json
kristal test --in plants.core.local.kristal.json
```

This ensures:

* property IDs exist in charters (P/R/S)
* value types match constraints
* claims reference valid Q-items
* deterministic ordering and stable IDs

---

## 7. Proposing the Change Upstream

The botanist submits:

* Kristal diff (claims + qitems changes)
* evidence refs / trace bundle (optional)
* validation report

See:

* `docs/en/contributors/propose_change.md`

---

## 8. Outcomes

The upstream maintainers may:

* accept the correction into `ctx:core`
* keep alternative classification as `ctx:contested`
* request improved evidence
* merge species additions into the main plant pack

---

## 9. Why Kristals Help Botanists

* Offline-first fieldwork support
* Domain focus reduces noise
* Mergeable expert contributions
* Explicit provenance and trust layers
* AI systems can ground answers in plant Kristals instead of general web text

