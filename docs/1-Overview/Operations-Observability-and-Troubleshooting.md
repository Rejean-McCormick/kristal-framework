# Operations: Observability & Troubleshooting

This page describes what to monitor and how to diagnose common failures in Kristal build, validation, distribution, activation, and query—without relying on internal implementation details.

## What to observe (minimum set)
### Build pipeline health
Track per stage:
- success/failure rate
- duration (median, p95/p99)
- queue/backlog depth
- retries and timeouts

Suggested stages:
- ingest / normalize
- resolve
- validate
- pack (runtime pack build)
- sign / seal
- publish

### Determinism & integrity signals
- % builds producing the same identity for the same input/config
- signature verification pass rate (by publisher / environment)
- hash mismatch events (should be near-zero)
- “declared integrity but unverifiable” count (should be zero)

### Distribution / activation health
- publish success rate
- activation success rate
- rollback events and reasons
- cache hit rates (if applicable)
- artifact fetch latency (p50/p95)

### Query service health (if applicable)
- request rate, error rate
- p50/p95 latency
- timeouts / circuit breaker opens
- pagination usage (page sizes, continuation tokens)
- capability negotiation failures

## Logging (recommended fields)
Use structured logs and ensure the following appear consistently:
- `tenant_id`
- `build_id` (correlation id)
- `exchange_id` / `kristal_id`
- `runtime_pack_id`
- `stage`
- `event` (START/END/ERROR)
- `duration_ms`
- `publisher_id` (if signed)
- `scope` (domain/subdomain) when sharding/federation is used

## Metrics (starter dashboard)
Build/validate:
- `build_success_rate{stage}`
- `build_duration_ms_p95{stage}`
- `validation_failures_total{reason}`
- `sentient_timeouts_total`

Integrity:
- `signature_verify_fail_total{publisher}`
- `hash_mismatch_total{artifact_type}`
- `missing_required_evidence_total`

Distribution:
- `publish_fail_total{reason}`
- `activation_fail_total{reason}`
- `rollback_total{reason}`
- `artifact_fetch_latency_ms_p95`

Query:
- `query_latency_ms_p95{operation}`
- `query_errors_total{code}`
- `pagination_tokens_issued_total`
- `capability_mismatch_total`

## Troubleshooting playbook (common issues)

### 1) “IDs change between builds”
Symptoms:
- same input snapshot + same config produces different IDs

Checks:
- confirm policies are recorded and unchanged
- check ordering / grouping / filtering settings
- verify determinism flags and config hashes
- ensure timestamps are not part of identity targets

### 2) “Validation fails after publish”
Symptoms:
- consumer rejects artifact after fetching

Checks:
- verify hash/signature material is present
- confirm validation reports exist and match the artifact IDs
- ensure the consumer is using the same profile versions
- check for missing files listed in manifests

### 3) “Signature verification fails”
Symptoms:
- reject due to invalid signature or unknown key

Checks:
- confirm correct key id / trust root configuration
- ensure policy/registry data is pinned and matches deployment
- check for revoked keys or wrong environment keys
- confirm what is signed vs what is excluded from the target

### 4) “Activation works but queries fail”
Symptoms:
- runtime pack activates but query errors/timeouts

Checks:
- confirm runtime pack references the correct exchange
- check query capability negotiation (supported operations)
- validate pagination usage and limits
- check indexes/filters policies (if relevant)

### 5) “Federation composes incorrectly”
Symptoms:
- unexpected conflicts / wrong precedence

Checks:
- verify every shard integrity + authority (fail-closed)
- confirm composition policy is deterministic and correctly versioned
- check pinned authority registry matches expected scope rules
- validate shard scopes are correctly declared

## Incident response (minimal template)
When an incident occurs:
- identify blast radius (tenants/scopes/publishers affected)
- freeze publishing if integrity is questionable
- roll back to last known-good activation if needed
- capture: build_id, exchange_id, runtime_pack_id, publisher_id, scope, error_code
- postmortem: root cause, prevention, follow-ups

## Next pages
- Operations: Release Strategy
- Workflow: Activate, Rollback & Downgrade
- FAQ