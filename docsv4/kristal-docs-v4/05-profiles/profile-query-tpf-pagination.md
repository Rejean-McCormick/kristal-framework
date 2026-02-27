# Profile: Query (TPF-like pagination)

## Status
Optional standardized profile (Kristal v4)

## Purpose

This profile defines an **offline-friendly, TPF-like pagination contract** for querying Kristal Runtime Packs. The intent is to provide:
- predictable, cacheable, low-bandwidth query behavior
- stable pagination semantics across implementations
- optional cardinality metadata to support planning and UI

This profile does **not** attempt to replicate full SPARQL semantics. Runtime Packs remain intentionally constrained and offline-executable.

## Scope

This profile specifies:
- request/response envelope for paginated queries
- cursor-token pagination semantics and stability guarantees
- ordering guarantees required for correct pagination
- optional cardinality metadata
- error and fail-closed semantics for declared capabilities

This profile does not specify:
- the full query language (see `04-query/query-contract.md`)
- network transport (HTTP vs local API), except for normative behaviors that must hold regardless of transport

## Conformance

An implementation claims this profile by including `profile-query-tpf-pagination@1` in the Runtime Pack Manifest `profiles[]` and declaring `query_contract` with:
- `query_contract.contract_id` (non-empty)
- `query_contract.supports_pagination = true`

If the implementation supports cardinality estimates, it MUST also declare:
- `query_contract.supports_cardinality_estimates = true`

If claimed, the implementation MUST meet the requirements below.

## Terminology

- **Page**: a bounded subset of results from a query.
- **Cursor token**: an opaque token allowing the client to fetch the next page.
- **Stable order**: a total order over the result set that does not change during pagination.

## Normative requirements

### 1) Capability declaration

1.1 A Runtime Pack that claims this profile MUST declare the query contract in the Runtime Pack Manifest:
- `query_contract.contract_id` (non-empty)
- `query_contract.supports_pagination = true`

1.2 If a pack claims this profile and a consumer requests pagination, the implementation MUST provide pagination behavior as specified here.

### 2) Stable ordering is mandatory for pagination

2.1 Paginated queries MUST have a stable total order over results.  
2.2 The order MUST be derived from the Runtime Pack’s recorded `policies.data_ordering`.  
2.3 Implementations MUST NOT paginate over an unstable or implementation-dependent ordering.

If a query’s semantics do not produce a stable order, the implementation MUST reject pagination for that query with an error (see §10).

### 3) Page size

3.1 Implementations MUST support a `page_size` parameter.  
3.2 Implementations SHOULD enforce a maximum page size to protect offline devices; if enforced, the maximum MUST be documented and MUST be discoverable via `capabilities` (see §8).

3.3 **Mapping to base query contract**  
The base query contract uses `limit`. When this profile is used:
- the effective `limit` MUST equal `page_size`
- clients SHOULD omit `limit` inside the `query` object
- if `query.limit` is present and differs from `page_size`, the implementation MUST return `INVALID_QUERY`

### 4) Cursor tokens

4.1 Cursor tokens MUST be opaque to clients.  
4.2 Cursor tokens MUST be deterministic for the same `(runtime_pack_id, query_hash, page_size, cursor_position)` and MUST NOT depend on wall-clock time.  
4.3 Cursor tokens MUST be bound to the specific Runtime Pack (`runtime_pack_id`). A token from one pack MUST NOT be valid on another pack.

4.4 Cursor tokens MUST include sufficient information (directly or indirectly) to resume iteration without re-scanning the full dataset, but the mechanism is implementation-defined.

### 5) Pagination API (logical contract)

Regardless of transport, the logical interface MUST accept:

- `query`: a query object defined in `04-query/query-contract.md` (with `limit` omitted or equal to `page_size` per §3.3)
- `page_size`: integer > 0
- `cursor`: optional opaque token

and MUST return:

- `results`: array of result rows/bindings
- `next_cursor`: null or opaque token
- `page_info`: metadata about the page

### 6) Response envelope (normative fields)

A conformant response MUST include:

- `runtime_pack_id`: sha256 hex
- `query_hash`: sha256 hex (hash of canonical query payload; see §7)
- `page_size`: integer
- `results`: array
- `next_cursor`: string or null
- `page_info`: object with:
  - `returned`: integer (count of results returned)
  - `has_more`: boolean

If the implementation supports cardinality estimates (and declares it in the manifest), responses MUST also include:
- `cardinality` (see §9)

### 7) Query hashing (for caching and comparability)

7.1 Implementations MUST compute:

`query_hash = sha256(JCS(query_payload))`

where `query_payload` includes:
- the query object (as provided, with `limit` normalized per §3.3)
- `page_size`
- any explicit ordering parameters (if and only if allowed by the base query contract)

`query_payload` MUST NOT include:
- `cursor`

7.2 Canonicalization for `query_hash` MUST use RFC 8785 (JCS) under the v3 core canonicalization identifiers:
- `canonicalization_profile = "kristal.v3:jcs-rfc8785"`
- `canonicalization_version = "1"`

### 8) Capabilities discovery

Implementations MUST provide a capabilities object discoverable via:
- an explicit API call, or
- a static file in the Runtime Pack (e.g., `query/capabilities.json`) referenced in `files[]`

Capabilities MUST include at least:
- supported query contract id(s)
- max page size (if enforced)
- whether cardinality estimates are supported

### 9) Cardinality metadata (optional but standardized)

If the implementation supports cardinality estimates, it MUST:
- set `query_contract.supports_cardinality_estimates = true` in the Runtime Pack manifest
- include `cardinality` in responses:

```json
"cardinality": {
  "type": "estimate",
  "value": 12345,
  "confidence": "low|medium|high"
}
````

Rules:

* `value` MUST be a non-negative integer.
* If exact cardinality is available, the implementation MAY use `"type": "exact"` and omit `confidence`.
* If cardinality is not supported, responses MUST NOT include `cardinality`.

### 10) Error handling

Errors MUST be structured and MUST include:

* `code`: stable string identifier
* `message`: human-readable message
* `details`: optional object

Required error codes:

* `UNSUPPORTED_PAGINATION` (query type cannot be paginated)
* `INVALID_CURSOR` (token invalid or not bound to this pack)
* `PAGE_SIZE_TOO_LARGE` (exceeds supported maximum)
* `INVALID_QUERY` (does not conform to base query contract or §3.3)
* `INTERNAL_ERROR` (unexpected failure)

### 11) Determinism and snapshot guarantees

11.1 For a given Runtime Pack, pagination MUST be consistent across repeated calls:

* using the same `cursor` token MUST yield the same subsequent results
* tokens MUST remain valid for the lifetime of the pack

11.2 Implementations MUST treat Runtime Packs as immutable snapshots. If the underlying data changes, it MUST be a new pack with a new `runtime_pack_id`.

### 12) Fail-closed behavior for declared capability

If a Runtime Pack Manifest claims this profile but the implementation cannot provide correct pagination behavior for a request that falls within the declared capability set, it MUST return an error (not partial/incorrect results).

## Non-normative guidance

* Prefer cursor designs that can resume without full scans (e.g., last-seen key + index position) but keep tokens opaque.
* Keep page_size defaults small for offline/low-memory environments.
* Use `(runtime_pack_id, query_hash, cursor)` as a primary cache key strategy for offline clients.

## Example request/response (non-normative)

Request:

```json
{
  "query": { "type": "spo", "s": "Q42", "p": "P31" },
  "page_size": 50
}
```

Response:

```json
{
  "runtime_pack_id": "aaaaaaaa...aaaa",
  "query_hash": "bbbbbbbb...bbbb",
  "page_size": 50,
  "results": [
    { "s": "Q42", "p": "P31", "o": "Q5" }
  ],
  "next_cursor": "opaque-token",
  "page_info": { "returned": 1, "has_more": true }
}
```

## Open questions

* Should the base query contract allow client-specified ordering, or require ordering to always follow `policies.data_ordering`?
* Should cursor tokens be required to be URL-safe for HTTP transports, or left transport-specific?

```

Alignment points used: runtime pack manifest `query_contract.*` fields and capabilities file pattern :contentReference[oaicite:0]{index=0} :contentReference[oaicite:1]{index=1}, and v3 canonicalization identifiers for JCS :contentReference[oaicite:2]{index=2}
```
