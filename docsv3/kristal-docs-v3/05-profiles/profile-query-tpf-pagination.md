````md
# Profile: Query (TPF-like pagination)

## Status
Optional standardized profile (Kristal v3)

## Purpose

This profile defines an **offline-friendly, TPF-like pagination contract** for querying Kristal Runtime Packs. The intent is to provide:
- predictable, cacheable, low-bandwidth query behavior
- stable pagination semantics across implementations
- optional cardinality metadata to support planning and UI

This profile does **not** attempt to replicate full SPARQL semantics. Runtime Packs remain intentionally constrained and offline-executable.

## Scope

This profile specifies:
- request/response envelope for paginated queries
- pagination tokens and stability guarantees
- ordering guarantees required for correct pagination
- optional cardinality metadata
- error and fail-closed semantics for declared capabilities

This profile does not specify:
- the full query language (see `04-query/query-contract.md`)
- network transport (HTTP vs local API), except for normative behaviors that must hold regardless of transport

## Conformance

An implementation claims this profile by including `profile-query-tpf-pagination@1` in the Runtime Pack Manifest `profiles[]` and declaring `query_contract` with:
- `supports_pagination = true`

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

If a query’s semantics do not produce a stable order, the implementation MUST reject pagination for that query with an error (see §8).

### 3) Page size

3.1 Implementations MUST support a `page_size` parameter.  
3.2 Implementations SHOULD enforce a maximum page size to protect offline devices; if enforced, the maximum MUST be documented and MUST be discoverable via `capabilities` (see §7).

### 4) Cursor tokens

4.1 Cursor tokens MUST be opaque to clients.  
4.2 Cursor tokens MUST be deterministic for the same (pack_id, query, page_size, cursor_position) and MUST NOT depend on wall-clock time.  
4.3 Cursor tokens MUST be bound to the specific Runtime Pack (`runtime_pack_id`). A token from one pack MUST NOT be valid on another pack.

4.4 Cursor tokens MUST include sufficient information (directly or indirectly) to resume iteration without re-scanning the full dataset, but the mechanism is implementation-defined.

### 5) Pagination API (logical contract)

Regardless of transport, the logical interface MUST accept:

- `query`: a query object defined in `04-query/query-contract.md`
- `page_size`: integer > 0
- `cursor`: optional opaque token

and MUST return:

- `results`: array of result rows/bindings
- `next_cursor`: null or opaque token
- `page_info`: metadata about the page

### 6) Response envelope (normative fields)

A conformant response MUST include:

- `runtime_pack_id`: sha256 hex
- `query_hash`: sha256 hex (hash of canonical query payload; see §6.2)
- `page_size`: integer
- `results`: array
- `next_cursor`: string or null
- `page_info`: object with:
  - `returned`: integer (count of results returned)
  - `has_more`: boolean

#### 6.2 Query hashing (for caching and comparability)

6.2.1 Implementations MUST compute `query_hash = sha256(JCS(query_payload))` where `query_payload` includes:
- the query object
- page_size
- any explicit ordering parameters (if allowed by the base query contract)
- MUST NOT include `cursor`

6.2.2 `query_hash` MUST use the v3 core canonicalization profile (`jcs-rfc8785@1`).

This enables cache keys and comparability across clients and implementations.

### 7) Capabilities discovery

Implementations MUST provide a capabilities object discoverable via:
- an explicit API call, or
- a static file in the Runtime Pack (e.g., `query/capabilities.json`) referenced in `files[]`

Capabilities MUST include at least:
- supported query contract id(s)
- max page size (if enforced)
- whether cardinality estimates are supported

### 8) Cardinality metadata (optional but standardized)

If the implementation supports cardinality estimates, it MUST:
- set `query_contract.supports_cardinality_estimates = true`
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
* If exact cardinality is available, the implementation MAY use `"type": "exact"` and omit confidence.

If cardinality is not supported, responses MUST NOT include `cardinality`.

### 9) Determinism and snapshot guarantees

9.1 For a given Runtime Pack, pagination MUST be consistent across repeated calls:

* using the same `cursor` token MUST yield the same subsequent results
* tokens MUST remain valid for the lifetime of the pack

9.2 Implementations MUST treat Runtime Packs as immutable snapshots. If the underlying data changes, it MUST be a new pack with a new `runtime_pack_id`.

### 10) Error handling

Errors MUST be structured and MUST include:

* `code`: stable string identifier
* `message`: human-readable message
* `details`: optional object

Required error codes:

* `UNSUPPORTED_PAGINATION` (query type cannot be paginated)
* `INVALID_CURSOR` (token invalid or not bound to this pack)
* `PAGE_SIZE_TOO_LARGE` (exceeds supported maximum)
* `INVALID_QUERY` (does not conform to base query contract)
* `INTERNAL_ERROR` (unexpected failure)

### 11) Fail-closed behavior for declared capability

If a Runtime Pack Manifest claims this profile but the implementation cannot provide correct pagination behavior for a request that falls within the declared capability set, it MUST return an error (not partial/incorrect results).

## Non-normative guidance

* Prefer cursor designs that can resume without full scans (e.g., last-seen key + index position) but keep tokens opaque.
* Keep page_size defaults small for offline/low-memory environments.
* Provide `query_hash` as a first-class cache key for clients (especially Konnaxion PWA caches).

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
```
