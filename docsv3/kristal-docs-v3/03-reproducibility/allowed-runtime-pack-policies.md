# Allowed Runtime Pack Policies (Kristal v3)

## Status
Draft (normative for v3 portability)

## Purpose
Kristal v3 aims to keep the **determinism surface area small** while still enabling high-performance offline execution. To avoid “compliant but incomparable” packs, v3 defines a **portable enumerated policy set** for Runtime Pack construction.

A v3 Runtime Pack MUST:
1. Select policy values from the allowed sets below, and  
2. Record those selections in the Runtime Pack manifest under `policy_selections`.

Anything outside these policies is either:
- a **non-normative implementation detail**, or
- an **optional profile extension** that MUST be explicitly declared and must not change core IDs unless included in the declared reproducibility surface.

## Normative language
The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are to be interpreted as normative requirements.

---

# 1. Policy recording requirements

## 1.1 Manifest structure (minimum)
The Runtime Pack manifest MUST include:

- `policy_selections.version` (e.g., `"v3.0"`)
- `policy_selections.ordering`
- `policy_selections.row_groups`
- `policy_selections.membership_filters`
- `policy_selections.bitmaps`
- `policy_selections.query_limits`

Implementations MAY include additional policy keys, but MUST NOT omit the keys above.

## 1.2 Determinism requirement
Given identical:
- inputs (same snapshots),
- compiler version + config hash,
- and the **same policy selections**,

then the resulting Runtime Pack MUST be reproducible (see `03-reproducibility/reproducibility-acceptance-tests.md`).

---

# 2. Ordering policies (portable)

Ordering is treated as an index. Runtime Pack builders MUST pick from the following ordering policies for the Parquet triples table.

## 2.1 Allowed orderings
The allowed base orderings are:

- `SPO`
- `SOP`
- `PSO`
- `POS`
- `OSP`
- `OPS`

## 2.2 Composite ordering sets
Implementations MAY build multiple ordered copies or indexes, expressed as a set:

- `ORDER_SET([SPO, POS])`
- `ORDER_SET([SPO])` (equivalent to single ordering)

If `ORDER_SET` is used, its ordering list MUST be recorded in a stable canonical order in the manifest (e.g., lexicographic).

## 2.3 Tie-breakers (mandatory)
For deterministic ordering, a tie-breaker chain MUST be declared:

- Primary keys: the chosen ordering (e.g., s then p then o)
- Tie-breaker: `claim_id` (or an equivalent stable statement identifier)
- Final tie-breaker: stable byte comparison of the serialized triple key (implementation-defined but deterministic)

---

# 3. Row-group sizing policies (portable)

Row-group policy impacts scan behavior and Parquet block indexes.

## 3.1 Allowed row-group modes
One of the following MUST be selected:

- `FIXED_ROWS(n)`  
- `FIXED_BYTES(n)`  
- `ADAPTIVE(target_rows, target_bytes, max_variance)`  

## 3.2 Constraints
- `n`, `target_rows`, `target_bytes`, and `max_variance` MUST be recorded.
- `ADAPTIVE` MUST be deterministic and MUST NOT depend on unstable runtime factors (e.g., CPU count).
- Builders SHOULD provide recommended defaults (non-normative).

---

# 4. Membership filter policies (portable)

Membership filters accelerate scans by quickly rejecting “not present” membership queries.
They are probabilistic, so v3 makes determinism requirements explicit.

## 4.1 Allowed filter families
One of the following families MAY be used:

- `BINARY_FUSE` (RECOMMENDED default)
- `XOR_FILTER` (OPTIONAL)

If no membership filter is used, the policy MUST be explicitly recorded as:

- `NONE`

## 4.2 Key spaces (portable)
The manifest MUST declare which key spaces are filtered. Allowed key spaces:

- `CLAIM_ID` (filter over statement identifiers)
- `SPO_KEY` (filter over a canonical triple key encoding)
- `SO_KEY` (optional: subject-object key encoding)
- `SP_KEY` (optional: subject-property key encoding)
- `PO_KEY` (optional: property-object key encoding)

Builders MUST declare the encoding for each key space and ensure it is deterministic.

## 4.3 Required parameters
For any non-NONE membership filter, the manifest MUST record:

- `family` (`BINARY_FUSE` or `XOR_FILTER`)
- `variant` (e.g., `3_WISE` or `4_WISE`, where applicable)
- `seed` (or a seed array if required by the family)
- `bits_per_key` OR an equivalent explicit sizing parameter
- `target_false_positive_rate` (if applicable)
- `key_spaces` (list of key spaces covered)

## 4.4 False positives (mandatory pruning rule)
Membership filters MAY return false positives. Therefore:

- A “membership hit” MUST NOT be treated as proof of membership.
- Implementations MUST deterministically prune false positives by validating against authoritative data (the triples table / index).
- No probabilistic acceptance is permitted at the semantic layer.

---

# 5. Bitmap index policies (portable)

Bitmaps accelerate joins and multi-value lookups.

## 5.1 Allowed bitmap format
- The allowed bitmap encoding is `ROARING`.

## 5.2 Run optimization
- `ROARING_RUN_OPT` SHOULD be enabled when it reduces size or improves scan speed.
- If enabled, it MUST be recorded as:

  - `run_optimization: ENABLED`

- If not enabled:

  - `run_optimization: DISABLED`

## 5.3 Container statistics (profile extension)
Recording container statistics (for cost modeling) is an OPTIONAL profile extension:

- `bitmap_container_stats: ENABLED|DISABLED`

If enabled, it MUST be declared as a profile and MUST NOT affect core identity unless included in the reproducibility surface.

---

# 6. Parquet-level policies (portable core vs profile)

## 6.1 Parquet schema invariants (portable core)
- Column names and logical types for the triples table MUST follow the v3 runtime schema definition (defined elsewhere).
- Sorting declaration MUST match the chosen ordering policy.

## 6.2 Bloom filters (optional profile)
Parquet Bloom filters MAY be enabled as an optional performance profile:

- `parquet_bloom_filters: ENABLED|DISABLED`

If enabled, the manifest MUST record:
- which columns have bloom filters,
- bloom filter sizing parameters,
- and any hashing seeds (if applicable).

Bloom filters MUST NOT be required for core conformance.

---

# 7. Query limits and strictness (portable)

Runtime Packs are not full SPARQL. To reduce cross-implementation divergence, v3 requires explicit query limits.

## 7.1 join1 cap policy (mandatory)
The manifest MUST declare:

- `join1_cap.default` (integer)
- `join1_cap.mode` in `{STRICT, TRUNCATE}`

### Semantics
- `STRICT`: exceeding the cap MUST produce a deterministic error code.
- `TRUNCATE`: exceeding the cap MUST return a deterministic truncated result set AND a deterministic warning flag.

## 7.2 Paging mode (mandatory)
The manifest MUST declare exactly one:

- `paging: CURSOR`
- `paging: OFFSET`

If `CURSOR`, cursor encoding MUST be deterministic and stable for the selected ordering policy.

---

# 8. Policy versioning

- `policy_selections.version` MUST be present.
- Any change to enumerations or semantics MUST bump the policy version.
- Implementations MUST reject unknown policy versions unless explicitly configured to allow them.

---

# 9. Summary (portable minimum)

A v3 Runtime Pack MUST record, at minimum:
- ordering policy (and tie-breakers),
- row-group policy,
- membership filter policy (or NONE) + parameters,
- bitmap policy + run optimization flag,
- query limits (join1 cap) + strictness,
- paging mode.

These policies define the **portable comparability surface** for Runtime Packs in v3.
