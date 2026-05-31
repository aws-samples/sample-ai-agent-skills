---
title: "A1 â€” Schema Mapping Creation"
description: "Diagnose schema mapping creation failures"
status: active
severity: HIGH
triggers:
  - "schema mapping creation failed"
  - "cannot create schema mapping"
  - "input source error"
owner: devops-agent
objective: "Resolve schema mapping creation issues"
context: "Schema mappings define how Glue Data Catalog table columns map to Entity Resolution attribute types (NAME, EMAIL_ADDRESS, etc.). Each mapping references a specific Glue database and table."
---

## Phase 1 â€” Triage

MUST:
- List schema mappings: `aws entityresolution list-schema-mappings`
- Verify Glue table exists: `aws glue get-table --database-name <db> --name <table>`
- Check column names match mapping configuration
- Verify IAM permissions: entityresolution:CreateSchemaMapping, glue:GetTable

SHOULD:
- Verify column data types are compatible with attribute types
- Check for duplicate attribute mappings

## Phase 2 â€” Remediate

MUST:
- Create schema mapping with correct table reference and attribute mappings
- Verify: `aws entityresolution get-schema-mapping --schema-name <name>`

## Common Issues

- symptoms: "Glue table not found"
  diagnosis: "Database or table name incorrect, or table in different region."
  resolution: "Verify Glue table with aws glue get-table. Ensure same region."

- symptoms: "Invalid attribute type mapping"
  diagnosis: "Column mapped to incompatible attribute type."
  resolution: "Use correct attribute types: NAME, ADDRESS, EMAIL_ADDRESS, PHONE_NUMBER, UNIQUE_ID."

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
