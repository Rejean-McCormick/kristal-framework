````md
# Operational guidance template (concept → problem → solution → when → pitfalls)

## Status
Template (non-normative)

## Purpose
A consistent structure for operational and implementation guidance across the Kristal ecosystem (Orgo, SenTient, Architect, Konnaxion).  
This template is **non-normative**: it does not define conformance requirements for Kristal artifacts.  
Use it to document failure paths, resilience, observability, and “least-worst” trade-offs.

---

## 1) Concept
**Name:**  
**One-liner:** (What it is)

**Applies to:** (Orgo / SenTient / Architect / Konnaxion / Compiler / Client)  
**Lifecycle stage:** (ingest / extract / resolve / validate / compile / export / distribute / query / render)

**Primary goal:** (e.g., availability, correctness, reproducibility, isolation)  
**Secondary goals:** (e.g., latency, cost, debuggability)

---

## 2) Problem
**What can go wrong?**  
- Failure modes: (timeouts, partial outputs, poisoned inputs, overload, nondeterminism, cache corruption, etc.)
- Blast radius: (single build, tenant, region, global distribution)
- User-facing impact: (missing packs, stale knowledge, incorrect output, blocked pipeline)

**How it fails today (if known):**  
- Symptoms:
- Triggers:
- Observed frequency:

---

## 3) Solution
**Approach:** (What you do and why)

**Mechanics (concrete):**
- Inputs:
- Outputs:
- State transitions:
- Retry policy:
- Backoff policy:
- Quarantine / DLQ policy:
- Timeouts and resource limits:
- Determinism guarantees: (what must remain identical across reruns)

**Data contracts / artifacts involved:**
- Exchange / Runtime Pack / Claim-IR / Resolved Claim-IR / Validation Report
- Which IDs must be carried end-to-end: (build_id, kristal_id, claim_id, evidence_id, tenant_id)

**Security considerations:**
- Trust roots / signature verification points
- Tenant isolation boundaries
- Downgrade/rollback rules

---

## 4) When to use
**Use when:**
- (e.g., dependency is flaky; workload spikes; offline distribution is required; RDF canonicalization is expensive)

**Do not use when:**
- (e.g., it would mask correctness failures; it would introduce nondeterministic outputs; it enlarges the standard surface area)

**Preconditions / dependencies:**
- (e.g., feature flags, queue support, local disk cache, stable clock, monotonic counters)

---

## 5) Pitfalls and trade-offs
**Pitfalls:**
- (e.g., silent degradation; retry storms; inconsistent caches; partial acceptance of probabilistic results)

**Trade-offs:**
- Correctness vs availability:
- Latency vs cost:
- Reproducibility vs performance:
- Isolation vs deduplication:

**Anti-patterns to avoid:**
- “Fail-open” when integrity is declared
- Writing unresolved ambiguity as resolved facts
- Letting operational metadata affect content hashes/IDs
- Making downstream generation introduce new facts

---

## 6) Observability
**Logs (structured fields required):**
- tenant_id
- build_id
- kristal_id
- stage
- attempt
- duration_ms
- outcome (success/fail/partial)
- error_code (stable)
- dependency (if external call)

**Metrics:**
- throughput (items/sec)
- success/failure rates
- retry rates
- queue depth / lag
- cache hit rate
- signature verification failures
- validation error counts by code

**Tracing:**
- Correlation IDs propagated across services
- Spans for: resolve, validate, compile, export, distribute, query, render

**Dashboards / alerts:**
- SLOs:
- paging thresholds:
- anomaly detection:

---

## 7) Runbook
**Detection:**
- What alerts fire?
- What dashboards to check first?

**Mitigation:**
- Step-by-step actions (ordered)
- Safe fallback modes (must remain deterministic)
- How to quarantine inputs / disable profile / roll back pack

**Recovery:**
- Criteria for returning to normal mode
- Post-incident checks (repro tests, signature checks, cache consistency)

**Postmortem notes (fields):**
- Root cause:
- Prevent recurrence:
- Action items:

---

## 8) Example (optional)
**Scenario:**  
**Before:**  
**After:**  
**Key outputs (sample log lines / fields):**
```json
{
  "tenant_id": "…",
  "build_id": "…",
  "kristal_id": "…",
  "stage": "resolve",
  "duration_ms": 1234,
  "outcome": "fail",
  "error_code": "SENTIENT_TIMEOUT"
}
````

---

## 9) Versioning and ownership

**Owner team:**
**Last updated:**
**Applies to versions:** (Kristal v3.x, Orgo version, etc.)
**Change log:**

* YYYY-MM-DD: …

```
```
