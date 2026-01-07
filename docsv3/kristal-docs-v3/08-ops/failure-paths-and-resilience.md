# Failure paths and resilience (non-normative)

## Status
Non-normative operational guidance (Kristal v3)

## Purpose

This document provides operational guidance for building and distributing Kristals reliably in the ecosystem:
- Orgo (workflow/orchestration)
- SenTient (resolution service)
- Kristal compiler (validation, Exchange finalize, Runtime Pack compile)
- Konnaxion (distribution/offline caching)
- Architect (deterministic rendering)

These patterns are **not** part of Kristal artifact schemas and do not affect conformance, except where core/spec requirements already mandate behaviors (e.g., fail-closed verification, “no compile on fail”).

## Principles

- **Fail fast, fail closed where integrity is involved.**
- **Preserve ambiguity rather than blocking pipelines** (SenTient resolution).
- **Quarantine poison inputs** (DLQ) instead of infinite retry or silent drop.
- **Bound resource usage** (timeouts/quotas) to protect offline and multi-tenant systems.
- **Prefer immutable snapshots** (Exchange + Runtime Pack releases), and treat new data as a new build.

## 1) Circuit breaker (SenTient resolution calls)

### Concept
A circuit breaker prevents repeated calls to an unhealthy dependency, reducing cascading failures.

### Problem it addresses
Resolution services can fail or degrade (timeouts, high latency, partial outages). Without protection, the build pipeline stalls, retries amplify load, and Orgo queues back up.

### Recommended approach
- Wrap all SenTient calls behind a circuit breaker per tenant and per resolver endpoint.
- Use a **funnel-friendly fallback**: if SenTient is unavailable, preserve unresolved ambiguity and continue if policy allows, or stop early with a clear failure reason.

### Suggested states
- **Closed:** normal operation
- **Open:** calls fail fast (no network call)
- **Half-open:** limited probe calls to test recovery

### Trigger thresholds (guidance)
- Open after N consecutive timeouts/errors within a short window.
- Remain open for a cool-down interval, then half-open probes.

### Output semantics
- If SenTient fails and the pipeline policy allows “unresolved preserved,” emit resolved-claim output with:
  - `status = "unresolved"`
  - `candidates = []` or last-known candidates
  - `warnings[]` with reason and correlation IDs

### Pitfalls
- Hiding failures: always log and surface circuit state changes.
- Global breakers: avoid one tenant taking down all tenants (use per-tenant scoping).

## 2) Dead letter queue (DLQ) for ingestion and build stages

### Concept
A DLQ stores messages/inputs that cannot be processed after bounded retries.

### Problem it addresses
“Poison inputs” (corrupt PDFs, malformed Claim-IR, pathological resolution cases) can block queues indefinitely if retried endlessly, or disappear if dropped.

### Recommended approach
- Use staged queues for: ingest → extract → resolve → validate → publish → distribute.
- Each stage has:
  - bounded retries
  - exponential backoff with jitter
  - DLQ on repeated failure

### DLQ payload requirements (guidance)
Include:
- `build_id`
- `tenant_id`
- `stage`
- `input_ref` (pointer to source or blob hash)
- `error_code` and a bounded error message
- `attempt_count`
- timestamps

### Operational loop
- DLQ items create Orgo Cases/Tasks automatically for triage.
- Triage outcomes:
  - fix input (requeue)
  - adjust policy (e.g., relax a non-core constraint)
  - mark as rejected with recorded reason

### Pitfalls
- DLQ as a graveyard: require ownership, SLAs, and dashboards.
- Infinite DLQ growth: apply retention + sampling for high-volume failures.

## 3) Timeouts and cancellation

### Concept
Every stage must have bounded execution time to prevent resource exhaustion and queue collapse.

### Recommended guidance (by stage)
- **Extraction (LLM/classical):** hard timeout; emit partial Claim-IR only if schema-valid and explicitly marked partial.
- **Resolution (SenTient):** hard timeout; preserve unresolved state; record warning.
- **Validation:** hard timeout; on timeout, treat as failure (no compile).
- **Pack compilation:** hard timeout; safe abort; no partial pack publication.
- **Distribution:** retry with backoff; never serve incomplete packs as “current.”

### Cancellation propagation
- If Orgo cancels a build, downstream workers MUST stop and emit a final build status record.
- Cancellation must not produce partially signed/published artifacts.

### Pitfalls
- Long tail: one slow job blocks worker pools.
- Partial publication: ensure atomic publish/commit semantics.

## 4) Quotas and rate limiting (multi-tenancy safety)

### Concept
Quotas bound per-tenant resource usage to protect system stability.

### Recommended quota dimensions
- Ingest volume (bytes/day, docs/day)
- Resolution calls (requests/min, tokens/day if applicable)
- Validation complexity (statements per build, references per statement)
- Build concurrency (active builds per tenant)
- Distribution bandwidth (MB/day per region)

### Enforcement guidance
- Enforce at Orgo orchestration boundary (admission control).
- Return structured errors with remediation hints.
- Provide “burst then throttle” behavior rather than hard drops when possible.

### Pitfalls
- Non-deterministic throttling affecting outputs: throttling MUST not change artifacts. It only changes scheduling.

## 5) Backpressure and queue hygiene

### Concept
When downstream is slow, upstream must slow down safely.

### Recommended approach
- Bounded queues with explicit backpressure signals to producers.
- Priority queues for:
  - critical rebuilds (security/rollback)
  - small builds (fast wins)
  - user-facing hotfixes

### Pitfalls
- Priority inversion: ensure large jobs cannot starve all others indefinitely.

## 6) Integrity-critical fail-closed points (must align with spec)

These are operational restatements of spec-level requirements:

- **Hash/signature verification is fail-closed** when declared.
- **No compile on validation failure**.
- **Atomic publish**: a release is either fully published and verifiable, or not published at all.
- **Immutable snapshots**: updates produce a new Exchange/Pack ID.

## 7) Release safety: canary / blue-green (distribution)

### Concept
Reduce blast radius when shipping new packs.

### Recommended approach
- Canary: deliver new pack to a small cohort/region first.
- Blue-green: keep previous pack live; switch traffic when canary passes.
- Always support rollback to last-known-good pack (subject to downgrade prevention policy).

### Health signals
- verification success rate
- query error rate
- pack download success and latency
- device cache hit rates (Konnaxion)

### Pitfalls
- Rollback that breaks integrity rules: ensure rollback is a signed, pinned prior release, not an unsigned artifact.

## 8) Observability: correlation IDs and structured logs

Required operational identifiers (guidance):
- `build_id` (end-to-end)
- `kristal_id` / `exchange_id`
- `runtime_pack_id`
- `claim_id` (Claim-IR / Exchange)
- `input_ref` (blob hash or source URI hash)
- `tenant_id`

Log events at:
- stage start/end
- circuit breaker state transitions
- retries and DLQ moves
- validation failure summaries (bounded)

## 9) Suggested minimal error taxonomy (guidance)

- `INGEST_PARSE_ERROR`
- `EXTRACT_SCHEMA_INVALID`
- `RESOLVE_TIMEOUT`
- `RESOLVE_DEP_UNAVAILABLE`
- `VALIDATION_FAILED`
- `VALIDATION_TIMEOUT`
- `COMPILE_FAILED`
- `PUBLISH_ATOMICITY_FAILED`
- `VERIFY_SIGNATURE_FAILED`
- `DISTRIBUTION_FAILED`
- `QUOTA_EXCEEDED`

## Appendix: recommended “least-worst” defaults

- Circuit breaker enabled for SenTient by default.
- DLQs enabled for every pipeline stage.
- Hard timeouts for every stage; validation timeout treated as failure.
- Quotas enforced at Orgo admission control.
- Canary rollout for packs before global promotion.

