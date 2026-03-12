# Workflows

This section describes the **typical ways Kristal is used end-to-end**, from building artifacts to operating them safely in production.

Each workflow page is written to be **action-oriented**: what you do, what artifacts you produce, and what “done” looks like.

## Core lifecycle (most common)
1. **Build & Validate**  
   Produce an Exchange (and optionally a Runtime Pack), run validation profiles, generate Validation Reports.

2. **Publish & Distribute**  
   Package artifacts for distribution (Konnaxion or another channel), attach signatures/metadata as needed.

3. **Activate & Serve**  
   Activate a specific version in an environment, enforce fail-closed checks, serve query interfaces.

4. **Operate & Update Safely**  
   Observe, debug, roll forward, or rollback/downgrade when needed.

## Extended workflows
- **Subsets (Recipes)**  
  Build smaller, deterministic Kristals for focused use cases (language, domain, time slice, tenant, etc.).

- **Federation & Curation**  
  Combine shards from multiple publishers without mutating them, apply authority/trust policy, publish a federation root.

## Workflow index
- [Workflow: Build & Validate](Workflow-Build-and-Validate)
- [Workflow: Publish & Distribute](Workflow-Publish-and-Distribute)
- [Workflow: Activate, Rollback & Downgrade](Workflow-Activate-Rollback-Downgrade)
- [Workflow: Subsets (Recipes)](Workflow-Subsets-Recipes)
- [Workflow: Federation & Curation](Workflow-Federation-and-Curation)