# Deterministic build rules (Kristal v4)

## Status
Normative (v3 core)

## Purpose

This document defines the **deterministic build requirements** for producing Kristal v4 artifacts, specifically:
- **Kristal Exchange** (canonical source of truth)
- **Kristal Runtime Pack** (derived offline-executable index)

The goal is **interoperability and reproducibility**: independent implementations MUST be able to rebuild artifacts with **bit-identical outputs** when given identical inputs and the same recorded policies.

## Scope

These rules apply to:
- the compilation pipeline stages that produce Exchange and Runtime Pack artifacts
- all content-addressed IDs and signatures associated with these artifacts
- manifests and files that claim deterministic outputs

These rules do **not** mandate a specific runtime performance strategy. They mandate that whatever strategy is used must be selected from **portable, enumerated policies** and must be **fully recorded** to enable reproducibility.

## Definitions

- **Deterministic build:** given the same input snapshot and the same recorded policies/parameters, the compiler produces identical outputs.
- **Reproducible build:** a third party can rerun the compiler using only the recorded manifests + referenced inputs and reproduce the exact outputs.
- **Input snapshot:** the complete set of inputs used for compilation, including Claim-IR, Resolved Claim-IR, validation rulesets, referenced evidence blobs (where applicable), and any subset-recipe inputs if used.
- **Portable policy:** an allowed, enumerated policy defined in `03-reproducibility/allowed-runtime-pack-policies.md`.
- **Hash target:** the JSON object that is canonicalized and hashed to produce a content-addressed ID, after applying required exclusions.

Normative keywords: MUST, MUST NOT, SHOULD, SHOULD NOT, MAY.

## Normative requirements

### 1) Determinism declaration

1.1 A Runtime Pack MUST declare whether it is deterministic.  
1.2 Kristal v4 Runtime Packs claiming v3 core conformance MUST set `build.deterministic = true` in the Runtime Pack Manifest.  
1.3 If an implementation cannot guarantee determinism for a given build, it MUST either:
- refuse to emit a v3 core-conformant pack, or
- emit a pack under a non-core profile that explicitly states non-deterministic behavior.

### 2) Canonicalization and hashing dependencies

2.1 Any JSON objects used for content-addressed IDs MUST be canonicalized using the declared canonicalization profile (RFC 8785 / JCS for v3 core).  
2.2 v3 core requires:
- `canonicalization_profile = "kristal.v3:jcs-rfc8785"`
- `canonicalization_version = "1"`

These values MUST be recorded by:
- every Exchange artifact, and
- every Runtime Pack Manifest (or its declared hashed-material projection).

2.3 For any content-addressed ID computation, the hashed material MUST:
- exclude the output ID field itself (e.g., `kristal_id`, `runtime_pack_id`), and
- exclude any `signatures` / attestations fields (wherever they appear),
and MUST otherwise follow the relevant hashed material definition for the artifact type (Exchange vs Runtime Pack).

2.4 If a manifest declares hashes/signatures and verification fails, consumers MUST fail-closed (see `01-core-spec/signatures-trust.md`).

### 3) Input snapshot is part of determinism

3.1 A deterministic build MUST be parameterized by an explicit input snapshot.  
3.2 The Runtime Pack Manifest MUST reference the Exchange it was compiled from (`exchange_ref.exchange_id`).  
3.3 The compiler configuration that affects output bytes MUST be hashed and recorded (`build.config_hash`).  
3.4 Any “implicit inputs” (defaults, environment variables, locale, system time) that affect output bytes MUST be eliminated or explicitly recorded under a portable, declared mechanism (e.g., policy fields or config hash inputs).

### 4) Ordering determinism (no unstable iteration)

4.1 The compiler MUST NOT rely on non-deterministic iteration order (e.g., hash maps without sorted traversal).  
4.2 All collections that affect output bytes MUST be processed in a deterministic order defined by an allowed ordering policy.

4.3 If an ordering policy is used, it MUST be recorded in the Runtime Pack Manifest under `policies.data_ordering`.

### 5) Portable, enumerated policies (small surface area)

5.1 Build-affecting behaviors MUST be chosen from the allowed policy set defined in `03-reproducibility/allowed-runtime-pack-policies.md`.  
5.2 If a required behavior is not covered by an allowed policy, the implementation MUST:
- propose a new policy for standardization, or
- emit a non-core profile pack that does not claim v3 core conformance.

This requirement exists to prevent “technically reproducible but practically incomparable” builds.

### 6) Parquet determinism (layout and encoding)

6.1 If Parquet is used, deterministic builds MUST:
- use a portable `data_ordering` policy
- use a portable `row_grouping` policy
- record Parquet encoding settings that affect file bytes (compression, dictionary encoding, stats, bloom filters if enabled)

6.2 The exact Parquet-related settings MUST be recorded in the Runtime Pack Manifest under `policies.parquet`.

6.3 If bloom filters are enabled, they MUST be treated as profile-bound unless explicitly included in v3 core allowed policies.

### 7) Membership filters and bitmaps determinism

7.1 If the pack includes membership filters (Bloom/Cuckoo/Xor/Binary-fuse), the kind and all parameters that affect bytes MUST be recorded in `policies.membership_filter` including (as applicable):
- seed(s)
- bits-per-key / fpp target
- fingerprint size / load factor
- hash function family / count
- any builder variant identifiers

7.2 If the pack includes bitmaps (e.g., Roaring), the bitmap format and any optimization steps that affect bytes MUST be recorded in `policies.bitmap`, including whether run optimization was applied.

7.3 Implementations MUST ensure membership filter and bitmap construction is deterministic (no random seeds unless explicitly recorded).

### 8) File inventory and integrity

8.1 The Runtime Pack Manifest MUST enumerate all files included in the pack under `files[]`, including:
- `path`
- `role`
- `sha256`
- `size_bytes`

8.2 The set of files and their hashes MUST be sufficient for consumers to verify the pack contents and integrity.

8.3 The file inventory MUST be deterministic:
- stable, deterministic ordering of `files[]` entries (e.g., lexical by `path`), and
- stable normalization of `path` values (no platform-specific separators).

### 9) Runtime Pack ID derivation

9.1 The `runtime_pack_id` MUST be derived from a canonical, deterministic representation of the pack’s hashed material.  
9.2 The hashed material MUST:
- exclude `runtime_pack_id` itself, and
- exclude any `signatures` / attestations fields (wherever they appear).

9.3 The hashed material MUST include, at minimum, a canonical projection covering:
- the referenced Exchange ID (`exchange_ref.exchange_id`)
- the deterministic build identity inputs (`build.config_hash`, `build.deterministic`)
- the declared canonicalization profile+version used for hashing
- the recorded portable policy selections (`policies.*`)
- the file inventory entries (each file’s `path` + `sha256` + `size_bytes` + `role`)

9.4 The exact hashed material projection (field set + ordering + exclusions) MUST be documented and test-vectorized.

### 10) Time, randomness, and environment constraints

10.1 The build MUST NOT incorporate non-deterministic sources (system time, nondeterministic PRNG, thread scheduling) into any bytes that are hashed or part of the pack outputs.  
10.2 `created_at` MAY reflect build time but MUST NOT affect any content-addressed IDs.  
10.3 If parallel compilation is used, the output MUST be equivalent to a deterministic serial order.

### 11) Validation as an acceptance gate

11.1 Runtime Pack compilation MUST depend on a successful deterministic validation step.  
11.2 If validation fails, the compiler MUST NOT produce:
- an Exchange commit, or
- a Runtime Pack that claims v3 conformance.

### 12) Reproducibility acceptance criteria

12.1 A v3 core-conformant build MUST pass the acceptance tests defined in:
`03-reproducibility/reproducibility-acceptance-tests.md`

12.2 At minimum, acceptance testing MUST verify:
- identical inputs + policies → identical output bytes for all pack files
- identical manifests (excluding non-hashed timestamps) and identical file inventories
- stable IDs across independent implementations

## Non-normative notes

- Keeping the policy surface area small is intentional: it encourages comparable builds and reduces ecosystem fragmentation.
- Advanced optimizations (e.g., Parquet bloom filters) should typically be gated behind explicit profiles unless they are proven portable and deterministic across toolchains.
- Operational resilience patterns (circuit breakers, DLQs, blue/green, canary) are described in `08-ops/` and are not part of artifact conformance.

## Open questions (to resolve in v3 finalization)

- Finalize the exact canonical “hashed material” projection for `runtime_pack_id` (normative field set + ordering + path normalization).
- Decide which Parquet knobs are permitted in v3 core vs profile-only.
- Decide whether `lexicographic_sop_asc` and `qid_pid_statement_id_asc` are sufficient ordering policies for core.