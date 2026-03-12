# Operations

This section is about running Kristal safely in real environments: releases, observability, troubleshooting, and compatibility.

The wiki stays practical and avoids repeating full specs. Where needed, we link to the technical docs.

---

## What “good operations” looks like

- Releases are **versioned**, **verifiable**, and **rollbackable**
- Activation is **atomic** (either the new artifact is fully active or nothing changes)
- Verification is **fail-closed** when integrity/trust signals are present
- You can answer: *what is running, where did it come from, and can we reproduce it?*
- Query behavior is stable and aligned with declared capabilities

---

## Core operational workflows

- **Release / rollout:** publish → validate published form → activate → monitor  
  See: [Release Strategy](Operations-Release-Strategy)

- **Rollback / downgrade:** revert to a known-good artifact quickly and safely  
  See: [Activate, Rollback & Downgrade](Workflow-Activate-Rollback-Downgrade)

- **Incident response:** detect → mitigate → verify integrity → restore service  
  See: [Observability & Troubleshooting](Operations-Observability-and-Troubleshooting)

---

## What to monitor (high level)

- Build success rate and stage durations
- Validation failures (top reasons, trends)
- Publish/activate success and rollback frequency
- Query latency and error rate by capability
- Cache/index health (if applicable)

---

## Compatibility

Operations should treat compatibility as a first-class concern:
- v3 vs v3.1 behaviors (activation, modules, patches)
- schema/profile versioning
- “what breaks cache / what breaks rollback”

See: [Compatibility (v3 / v3.1)](Operations-Compatibility)

---

## Next pages

- [Release Strategy](Operations-Release-Strategy)
- [Observability & Troubleshooting](Operations-Observability-and-Troubleshooting)
- [Compatibility (v3 / v3.1)](Operations-Compatibility)