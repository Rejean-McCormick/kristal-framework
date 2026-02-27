# Allowed Runtime Pack Policies (Kristal v4)

## Status
Draft (normative for v3 portability)

## Purpose
Kristal v4 aims to keep the **determinism surface area small** while still enabling high-performance offline execution. To avoid “compliant but incomparable” packs, v3 defines a **portable enumerated policy set** for Runtime Pack construction.

A v3 Runtime Pack MUST:
1. Select policy values from the allowed sets below, and
2. Record those selections in the Runtime Pack manifest under `policies`.

Anything outside these policies is either:
- a **non-normative implementation detail**, or
- an **optional profile extension** that MUST be explicitly declared and must not change core IDs unless included in the declared reproducibility surface.

## Normative language
The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are to be interpreted as normative requirements.

---

# 1. Policy recording requirements

## 1.1 Manifest structure (minimum)
The Runtime Pack manifest MUST include a `policies` object with (at minimum):

- `policies.data_ordering`
- `policies.row_grouping`
- `policies.membership_filter`
- `policies.bitmap`

The manifest MAY also include:

- `policies.parquet`

Implementations MAY include additional policy keys only if the Runtime Pack manifest schema permits, but MUST NOT omit the keys above.

## 1.2 Determinism requirement
Given identical:
- inputs (same snapshots),
- compiler version + config hash,
- and the **same recorded policies**,

then the resulting Runtime Pack MUST be reproducible (see `03-reproducibility/reproducibility-acceptance-tests.md`).

---

# 2. Data ordering policies (portable)

Ordering is treated as an index. Runtime Pack builders MUST pick from the following ordering policies for the primary triples store.

## 2.1 Allowed ordering policies
Exactly one of the following MUST be selected in `policies.data_ordering.policy`:

- `qid_pid_statement_id_asc`
- `lexicographic_sop_asc`
- `none`

## 2.2 Semantics (normative)
- `qid_pid_statement_id_asc` MUST define a total order where the primary sort is by subject (QID-like), then predicate (PID-like), then a stable statement identifier (e.g., `statement_id`) as the tie-breaker.
- `lexicographic_sop_asc` MUST define a total order by stable byte-wise lexicographic comparison of an implementation-defined *canonical* triple key encoding whose fields are `(s, o, p)` in that order.
- `none` indicates the pack does not declare a portable total order. Packs that claim pagination profiles SHOULD NOT use `none`.

## 2.3 Tie-breakers (mandatory)
If the chosen ordering can produce ties under its primary key(s), implementations MUST apply a deterministic tie-breaker chain that produces a total order:
1. The ordering policy’s primary keys
2. A stable statement identifier when available
3. Stable byte comparison of a canonical serialized key (implementation-defined but deterministic)

---

# 3. Row-group sizing policies (portable)

Row-group policy impacts scan behavior and Parquet block indexes.

## 3.1 Allowed row-group modes
Exactly one of the following MUST be selected in `policies.row_grouping.policy`:

- `fixed_rows_100k`
- `fixed_rows_1m`
- `fixed_bytes_128mb`

## 3.2 Determinism constraints
- The selected policy MUST be applied deterministically.
- Implementations MUST NOT vary row groups based on unstable runtime factors (e.g., CPU count).

---

# 4. Membership filter policies (portable)

Membership filters accelerate scans by quickly rejecting “not present” membership queries. They are probabilistic, so v3 makes determinism requirements explicit.

## 4.1 Allowed filter kinds
Exactly one of the following MUST be selected in `policies.membership_filter.kind`:

- `none`
- `bloom`
- `cuckoo`
- `xor8`
- `xor16`

## 4.2 Required parameters
For `kind = none`, no additional parameters are required.

For `kind != none`, the manifest MUST record all required parameters for that kind:

- `bloom`:
  - `seed`
  - `bits_per_key`
  - `hash_functions`

- `cuckoo`:
  - `seed`
  - `fingerprint_bits`
  - `load_factor`

- `xor8` / `xor16`:
  - `seed`
  - `bits_per_key`

## 4.3 False positives (mandatory pruning rule)
Membership filters MAY return false positives. Therefore:
- A “membership hit” MUST NOT be treated as proof of membership.
- Implementations MUST deterministically prune false positives by validating against authoritative data (the triples store / index).
- No probabilistic acceptance is permitted at the semantic layer.

---

# 5. Bitmap index policies (portable)

Bitmaps accelerate joins and multi-value lookups.

## 5.1 Allowed bitmap formats
The allowed bitmap formats are:

- `roaring`
- `roaring_run_optimized`

Selected via `policies.bitmap.format`.

## 5.2 Run optimization
- If run optimization is enabled, it MUST be recorded as: `policies.bitmap.run_optimize = true`
- If not enabled, it MUST be recorded as: `policies.bitmap.run_optimize = false`

## 5.3 Container statistics (optional profile extension)
Recording bitmap container statistics (for cost modeling) is an OPTIONAL profile extension. If enabled, it MUST be declared as a profile and MUST NOT affect core identity unless included in the reproducibility surface.

---

# 6. Parquet-level policies (portable core vs profile)

## 6.1 Parquet schema invariants (portable core)
- Column names and logical types for the triples table MUST follow the v3 runtime schema definition.
- Sorting declaration MUST match the chosen data ordering policy, if applicable.

## 6.2 Allowed Parquet policy fields
If `policies.parquet` is present, it MUST use only allowed enumerations:

- `compression` in `{zstd, snappy, gzip, none}`
- `dictionary_encoding` in `{true, false}`
- `statistics` in `{none, page, rowgroup}`

## 6.3 Bloom filters (optional profile)
Parquet Bloom filters MAY be enabled as an optional performance profile, recorded as:

- `policies.parquet.bloom_filters.enabled = true|false`

If enabled and the manifest schema permits additional details, the pack SHOULD record:
- which columns have bloom filters, and
- sizing/hash parameters needed to reproduce them.

Bloom filters MUST NOT be required for core conformance.

---

# 7. Query contract linkage (portable)

This document specifies build-affecting Runtime Pack construction policies. Runtime query semantics are specified by `04-query/query-contract.md` and any declared query profiles.

A Runtime Pack SHOULD declare `query_contract` in the manifest (e.g., `contract_id`, and capability flags) and MUST ensure any declared pagination behavior derives its stable order from `policies.data_ordering`.

---

# 8. Versioning
- Policy enumerations are versioned by the Runtime Pack manifest schema version.
- Any change to enumerations or semantics MUST bump the relevant schema/profile version.
- Implementations MUST reject unknown policy values and unknown schema/profile versions unless explicitly configured to allow them.

---

# 9. Summary (portable minimum)

A v3 Runtime Pack MUST record, at minimum:
- `policies.data_ordering`
- `policies.row_grouping`
- `policies.membership_filter` (including required parameters for the selected kind)
- `policies.bitmap` (including `run_optimize`)
- (optionally) `policies.parquet`

These policies define the **portable comparability surface** for Runtime Packs in v3.