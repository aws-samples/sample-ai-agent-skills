---
title: "A2 â€” Schema Mapping Validation"
description: "Diagnose schema mapping validation and compatibility issues"
status: active
severity: MEDIUM
triggers:
  - "schema mapping validation error"
  - "attribute mapping mismatch"
  - "incompatible schema"
owner: devops-agent
objective: "Resolve schema mapping validation problems"
context: "Schema mappings must have valid attribute type assignments and reference existing Glue tables. Matching workflows validate that input schema mappings are compatible."
---

## Phase 1 â€” Triage

MUST:
- Get schema mapping: `aws entityresolution get-schema-mapping --schema-name <name>`
- Verify all mapped columns exist in the Glue table
- Check attribute types are valid Entity Resolution types
- Verify UNIQUE_ID attribute is mapped (required for matching)

SHOULD:
- Check that matching workflow input schemas are compatible
- Verify no duplicate UNIQUE_ID values in source data

## Phase 2 â€” Remediate

MUST:
- Fix attribute type assignments to use valid types
- Ensure UNIQUE_ID is mapped for each input schema
- Update schema mapping if Glue table columns changed

## Common Issues

- symptoms: "Missing UNIQUE_ID attribute"
  diagnosis: "Schema mapping doesn't include a UNIQUE_ID attribute."
  resolution: "Map a column with unique record identifiers to the UNIQUE_ID attribute type."

- symptoms: "Schema mappings incompatible for matching"
  diagnosis: "Input schemas don't share common attribute types for matching."
  resolution: "Ensure input schemas map columns to the same attribute types (e.g., both have EMAIL_ADDRESS)."

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | MEDIUM — Entity Resolution processes batch matching jobs on customer data |
| Data Risk | CRITICAL — Processes PII including names, addresses, emails, and phone numbers |
| Recovery Complexity | LOW — Jobs can be re-run but consume time and compute resources |
| Blast Radius | MEDIUM — Incorrect matching can propagate bad data to downstream systems |

## Escalation Conditions

- Matching workflow producing false positives linking unrelated customer records
- Schema mapping changes causing data misalignment in production workflows
- Provider service integration failures blocking third-party identity resolution
- Job execution failures on large datasets exceeding service quotas
- Output data containing unexpected PII exposure or cross-contamination
- ML matching confidence scores dropping significantly between job runs

## Data Sensitivity

- Input datasets contain PII: names, addresses, emails, phone numbers, unique identifiers
- Match output reveals entity relationships that may be privacy-sensitive
- Schema mappings expose data structure and attribute classification
- Provider service configurations may contain third-party API credentials
- ID mapping results link identities across systems — treat as highly sensitive

## PROHIBITED Actions

- NEVER suggest deleting schema mappings that are referenced by active matching workflows
- NEVER modify matching rules on production workflows without testing on sample data first
- NEVER expose raw PII from input or output datasets in logs or troubleshooting output
- NEVER change ML matching confidence thresholds without stakeholder approval
- NEVER delete matching workflow output without confirming downstream consumers are notified
- NEVER share provider service credentials or API keys in plain text

## Phase 3 — Rollback

- If schema mapping change broke matching: revert to previous schema mapping version and re-run workflow
- If matching rules produced bad results: restore previous rule configuration and invalidate bad output
- If ML matching confidence was changed: revert threshold and re-run affected jobs
- If output configuration was modified: restore previous S3 output path and format settings
- If provider service integration broke: revert to previous provider configuration and verify connectivity
- Document all rollback actions with workflow ARN, job ID, and affected record counts
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
