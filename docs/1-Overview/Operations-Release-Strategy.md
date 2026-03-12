# Operations: Release Strategy

This page describes how to **ship Kristal artifacts safely** across environments (dev → staging → prod), with clear rollback paths and minimal ambiguity for consumers.

The goal is to make every release:
- **Traceable** (what version is live, where did it come from?)
- **Deterministic** (same inputs/config ⇒ same outputs)
- **Reversible** (rollback/downgrade is a first-class operation)
- **Verifiable** (fail-closed when integrity/trust is declared)

## Release units
A “release” usually activates one of these:
- **Exchange** (the canonical Kristal artifact)
- **Runtime Pack** (serving-optimized packaging for query)
- **Federation Root** (if you publish curated federations)
- **Authority Registry** (pinned trust policy data)

In most deployments, **Runtime Packs** are what you activate for serving, while Exchanges remain the canonical reference.

## Environments and promotion
Typical promotion path:
1. **Dev**: frequent builds, relaxed gating, quick iteration
2. **Staging**: production-like configs, strict validation, load tests
3. **Prod**: signed artifacts, fail-closed verification, strict rollouts

Promotion should be *artifact-based*:
- Promote by referencing the same content-addressed ID across environments
- Avoid “rebuild per environment” unless you also publish the build evidence and treat it as a new artifact

## Recommended rollout patterns
### 1) Canary → expand
- Activate the new pack for a small % of traffic
- Watch query latency/error rates and validation failures
- Expand gradually until full rollout

### 2) Blue/green
- Run two complete stacks side-by-side (old/new)
- Switch traffic at the router layer
- Rollback is an instantaneous flip

### 3) Pin-and-freeze (for critical consumers)
- Pin a specific artifact/version for a consumer segment
- Only move when explicitly approved
- Useful when clients must remain fully reproducible for audits

## What must be recorded for each release
At minimum:
- Activated **artifact IDs/hashes**
- Build correlation IDs (build_id, config_hash)
- Validation profile set + outcome (links to Validation Reports)
- Signing/trust material (who signed, which key ID, which authority class)
- Environment metadata (where activated, when, by whom)

## Rollback and downgrade
A safe strategy requires two operations:
- **Rollback**: revert to the previously activated artifact/version
- **Downgrade**: revert to an older compatible major/minor version (policy-driven)

Operational rules:
- Rollback should not require a rebuild (use already-published artifacts)
- Downgrade should be treated like a new rollout (staging + validation)
- Always log activation changes with correlation IDs

## Validation gating (practical)
Common gating rules:
- Do not publish/activate if required validation profiles failed
- Do not activate if signatures/integrity checks fail (when declared)
- Require additional validations for “recognized authority” publishers

## Monitoring during releases
Track at least:
- Activation success/failure
- Query error rate and latency (p50/p95)
- Validation failure reasons (if executed at runtime)
- Cache/index warm-up time
- Rollback events and causes

## Tech details
- Ops release strategies: `kristal-docs-v4/08-ops/release-strategies.md`
- Rollback/downgrade policy: `kristal-docs-v4/07-security/downgrade-rollback-policy.md`
- Distribution/activation contract: `kristal-docs-v4/06-integration/konnaxion-distribution-contract.md`