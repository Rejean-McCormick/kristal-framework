# Query Basics

This page explains **how clients query a Kristal** at a practical level: what you can ask, what you get back, and how to think about capabilities.

Kristal supports query through a **declared contract** (the server/runtime tells you what it supports), so clients can adapt across implementations.

## What you can do (at a high level)
Common query tasks:
- Look up an entity (by ID) and retrieve its statements
- Traverse relationships (neighbors, edges, properties)
- Filter by basic constraints (property, language, scope)
- Page through results deterministically
- Use server-declared capabilities for performance-friendly access patterns

## The “contract-first” model
Every query endpoint/runtime pack should expose a **Query Contract** describing:
- Supported operations (and versions)
- Pagination model(s)
- Limits/caps (max results, max joins, timeouts)
- Optional features (e.g., estimates, additional projections)

Clients should:
1. Fetch/read the contract
2. Choose an operation that’s supported
3. Execute within declared caps
4. Handle paging until completion (or until the client’s own limits)

## Typical request/response shape (conceptual)
Even if APIs differ (HTTP, local embedding, SDK), most query flows follow the same pattern:

1. **Select operation** (e.g., “entity lookup”, “TPF-like page”, “join1”)
2. **Provide parameters**
   - identifiers (entity IDs, property IDs)
   - optional filters (language, scope)
   - page token / cursor (if paging)
3. **Receive results**
   - data payload
   - next cursor / token (if more data)
   - optional metadata (timing, partial results, caps hit)

## Paging and determinism
Two rules keep results stable and debuggable:
- Paging must be **repeatable** for a fixed Kristal version + operation + parameters.
- The server should return a **cursor/token** that is valid for the same activated artifact/version.

If a client sees different results across pages, it should treat that as a **contract violation** or a sign the underlying activated version changed.

## Capabilities and fallbacks
Capabilities let clients be efficient without assuming too much:
- If “join1” is supported, use it.
- If not, fall back to simpler operations (e.g., repeated lookups or TPF-like paging).
- If estimates are available, use them to size work; if not, behave conservatively.

## Troubleshooting tips
If queries behave unexpectedly:
- Confirm which Kristal version is activated (IDs/hashes)
- Confirm the Query Contract version and caps
- Check whether paging tokens are being reused correctly
- Check server logs/correlation IDs (ops)

## Tech details
- Query contract: `kristal-docs-v4/04-query/query-contract.md`
- Pagination profile: `kristal-docs-v4/05-profiles/profile-query-tpf-pagination.md`
- Runtime Pack manifest fields related to query: `kristal-docs-v4/02-schemas/runtime-pack-manifest.schema.json`