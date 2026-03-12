# Query: Pagination & Capabilities

This page explains how Kristal query endpoints expose **capabilities** (what they support) and **pagination** (how you page through results) in a predictable, client-friendly way.

The wiki stays at the “contract level” (how to use it), not the detailed technical spec.

---

## 1) Capabilities (what a server can do)

A Kristal query service should advertise:
- Which query modes it supports (e.g., basic lookups, pattern queries, joins)
- Limits (page size, max results, timeouts)
- Optional features (cardinality estimates, ordering guarantees, filters)

### Why it matters
Clients can:
- Adapt automatically to different deployments
- Avoid requesting unsupported operations
- Use consistent fallback behavior

### Typical capability fields (conceptual)
- **Supported operations**: list of named operations
- **Limits**: max page size, max time per query, max rows
- **Estimates**: whether the server can return approximate counts
- **Stability**: ordering guarantees, consistency model

---

## 2) Pagination (how to page results)

Kristal services should use a **token-based** pagination model:
- The server returns a `next_page_token` when more results exist
- The client passes the token back to continue

### Recommended client behavior
- Treat tokens as opaque
- Don’t try to decode or modify them
- Cache tokens only short-term (they may expire)

### Server-side expectations
- Tokens should be deterministic for a given (query, version, ordering policy)
- If ordering is not guaranteed, it must be declared in capabilities

---

## 3) Page size & limits

Clients should:
- Start with a moderate page size
- Back off if the server reports throttling or timeouts
- Respect server-declared caps

Servers should:
- Enforce caps consistently
- Return clear error codes for “page too large” or “unsupported capability”

---

## 4) Cardinality estimates (optional)

Some deployments can return approximate counts (for UI and planning):
- Estimated total results
- Estimated remaining results

If estimates are supported, they should be clearly marked as approximate.

---

## 5) Practical examples (high level)

### Pattern query paged
1. Client submits query with `page_size`
2. Server returns results + `next_page_token`
3. Client repeats until token is absent

### Capability-aware fallback
1. Client reads capabilities
2. If `join1` unsupported, client switches to a simpler query plan
3. Client retries with supported operations

---

## Related pages
- [[Query-Basics]]
- [[Query]]
- [[Workflow-Activate-Rollback-Downgrade]]
- [[Operations-Compatibility]]

## Tech details (links)
- Query contract: `04-query/query-contract.md`
- Profile: `05-profiles/profile-query-tpf-pagination.md`