
# Use Case: Historian (Evidence-Aware Knowledge Packs)

## Status
DRAFT

## 1. Scenario

A historian wants to build and maintain an evidence-aware knowledge base about a topic such as:
- a time period (e.g., 19th-century Quebec),
- a person/network (political actors, writers, institutions),
- an event sequence (wars, treaties, migrations),
- a corpus of primary sources (letters, newspapers, archives).

They need:
- structured claims (who/what/when/where),
- careful provenance,
- the ability to represent disagreements and uncertainty,
- offline workflows for archival research,
- mergeable collaboration between researchers.

Kristals support this by packaging a bounded, queryable graph that can be sliced, edited, and versioned.

---

## 2. Goals

- **Offline work**: archives, restricted reading rooms, travel
- **Provenance-first**: claims tied to sources and locators
- **Multiple interpretations**: contested claims are first-class
- **Traceable compilation**: AI-assisted extraction without losing auditability
- **Reproducible releases**: stable IDs and deterministic builds
- **Collaboration**: merge contributions from multiple historians

---

## 3. Downloading a History Kristal

### 3.1 Curated domain pack
Example:

```bash
kristal pull kristal://history.quebec.19c --out ./kristals/
````

The pack includes:

* relevant Q-items (people, places, institutions, events)
* claims describing relations, dates, locations
* charters defining any P/R/S properties used
* optional indexes for quick lookup

### 3.2 Query-built slice (topic-centered)

Example: build a pack around a seed entity (a person or event):

```bash
kristal pull --seed Q<person> --closure bidirectional --depth 2 --out person.network.kristal.json
```

---

## 4. Typical Queries

### 4.1 Timeline extraction (events by time)

Retrieve event dates:

```sparql
SELECT ?event ?date WHERE {
  ?event P31 Q<event> .
  ?event P585 ?date .
}
```

### 4.2 Network graph around a person

```sparql
SELECT ?p ?o WHERE { Q<person> ?p ?o . }
```

### 4.3 Find sources supporting a claim

At the Kristal level, references are stored as stubs. The historian can query for claims that point to a given `source_ref`:

```sparql
SELECT ?s ?p ?o WHERE {
  ?s ?p ?o .
  FILTER(source_ref = "source://sha256:<hash>")
}
```

(Exact syntax depends on the local engine; concept is required.)

---

## 5. Editing & Evidence Handling

### 5.1 Add a claim with precise provenance

Historians typically add:

* `references[]` with document locator (page/offset)
* `trace_refs[]` for extraction notes or AI runs

Example concept:

* “Person A attended meeting B on date D” with a citation to an archival record.

### 5.2 Represent uncertainty and competing accounts

Historical facts often conflict. Kristals allow:

* different `status` values (`asserted`, `contested`, `needs_review`)
* contexts (core vs contested layers)
* multiple claims that disagree, without overwriting

Example:

* two different birth dates from different sources, both stored and tagged.

### 5.3 AI-assisted extraction (optional)

If using AI to extract claims from documents:

* keep raw outputs in trace bundles
* store only minimal references in Kristal core
* require human review for promotion into `ctx:core`

---

## 6. Validation and Testing

Before publishing or sharing:

```bash
kristal validate --in history.topic.local.kristal.json
kristal test --in history.topic.local.kristal.json
```

Key checks for historians:

* time value parsing is correct
* sources are linked by stable hashes/locators
* claims do not silently overwrite contested alternatives
* deterministic ordering ensures stable review diffs

---

## 7. Publishing a Research Release

Historians often publish snapshots (versions) tied to papers or datasets:

* `content_version`: “2025.12.19-archive-scan-2”
* `release_channel`: “stable” for reviewed content, “beta” for working sets

Lineage metadata should identify:

* which Kristals were merged
* which slices were derived

---

## 8. Why Kristals Help Historians

* structured timeline and network queries offline
* provenance built into the workflow (without bloating)
* explicit support for contested interpretations
* reproducible versions for citations in academic work
* collaboration through mergeable, reviewable changes



