---
title: "B1 â€” Matching Workflow"
description: "Diagnose matching workflow creation and configuration issues"
status: active
severity: HIGH
triggers:
  - "matching workflow error"
  - "cannot create matching workflow"
  - "workflow configuration issue"
owner: devops-agent
objective: "Resolve matching workflow creation and configuration problems"
context: "Matching workflows define the matching strategy (rule-based or ML), input schema mappings, output location, and IAM role. They orchestrate the matching process across input datasets."
---

## Phase 1 â€” Triage

MUST:
- List workflows: `aws entityresolution list-matching-workflows`
- Get workflow: `aws entityresolution get-matching-workflow --workflow-name <name>`
- Verify input schema mappings exist and are valid
- Check IAM role has required permissions
- Verify output S3 location

SHOULD:
- Check matching technique configuration (rule-based vs ML)
- Verify resolution type (MATCH_ID_MAPPING or RULE_MATCHING)

## Phase 2 â€” Remediate

MUST:
- Fix workflow configuration (input schemas, role, output)
- Verify IAM role trust policy includes `entityresolution.amazonaws.com`
- Ensure role has glue:GetTable, glue:GetDatabase, s3:PutObject permissions

## Common Issues

- symptoms: "Input schema mapping not found"
  diagnosis: "Referenced schema mapping was deleted or name is incorrect."
  resolution: "Verify schema mapping exists with list-schema-mappings."

- symptoms: "IAM role cannot be assumed"
  diagnosis: "Role trust policy doesn't include entityresolution.amazonaws.com."
  resolution: "Update role trust policy to allow entityresolution.amazonaws.com to assume it."

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
