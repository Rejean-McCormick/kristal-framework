# Artifact: Validation Report

## What it is
A **Validation Report** is the artifact that records *what was validated*, *which rules/profiles were applied*, and *what the outcome was* for a given build/output. It’s the traceable proof that an Exchange (or shard) met the expected checks at the time it was produced.

## When you’ll see it
Typically produced during the **Build & Validate** workflow, before publishing/distribution.

Common triggers:
- CI / pipeline gating (“only publish if validation passes”)
- Publisher policy (“recognized authority requires extra checks”)
- Federation consumption (“accept shard only if certain validations passed”)

## What it’s used for
- **Fail-closed verification** when a manifest declares that certain validations are required
- Debugging and audit: *why did this build pass/fail?*
- Federation policy: deciding whether a shard is acceptable under a trust/authority rule-set

## What it contains (human view)
At a high level, a Validation Report answers:

- **Target**: which artifact/build the report refers to (IDs + build correlation)
- **Validator tooling**: what validator ran, version/config
- **Profiles executed**: which validation profiles were applied (e.g., SHACL, ShEx, schema checks, RDFC)
- **Results**: pass/fail + summary counts + key errors/warnings
- **Integrity**: hashes/signatures (optional but recommended for distribution/archival)

## Who consumes it
- Publishers (to gate publishing)
- Distributors/activators (to enforce “only verified artifacts go live”)
- Federation consumers (to enforce authority policies and trust requirements)
- Operators (to diagnose failures, regressions, mismatched configs)

## Relationship to other artifacts
- **Exchange / Shard Manifest**: may include references to one or more Validation Reports as evidence.
- **Runtime Pack**: may be produced only after validations pass (policy-dependent).
- **Authority Registry / Policies**: can require specific validations for specific scopes.

## Practical guidance
- Treat Validation Reports as **first-class evidence**: store them alongside the artifacts they validate.
- If you have a strict pipeline, make validation references **mandatory** for publish/activate steps.
- Keep reports small: store detailed logs as external references when possible, but keep a reliable summary in the report.

## Tech details
- JSON schema: `kristal-docs-v4/02-schemas/validation-report.schema.json`
- Example: `kristal-docs-v4/10-examples/validation-report.example.json`
- Related: `kristal-docs-v4/01-core-spec/signatures-trust.md`