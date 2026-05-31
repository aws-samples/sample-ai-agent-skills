---
title: "D1 — Seed Audience Errors"
description: "Diagnose seed audience data quality, format, and size issues"
status: active
severity: HIGH
triggers: ["seed audience error", "seed data invalid", "seed too small", "seed format error"]
owner: devops-agent
objective: "Resolve seed audience data issues for successful audience generation"
context: "Seed audiences define the target segment for lookalike audience generation. They must meet minimum size requirements and conform to expected schema formats."
---

## Phase 1 — Triage

MUST:
- Verify seed audience S3 location is accessible: `aws s3 ls s3://<seed-bucket>/<prefix>/`
- Check seed audience file size and record count: `aws s3api head-object --bucket <bucket> --key <key>`
- Get audience generation job error details: `aws cleanroomsml get-audience-generation-job --audience-generation-job-arn <arn>`

SHOULD:
- Validate seed audience schema matches training dataset schema
- Check for empty files or files with only headers

MAY:
- Sample seed audience data to verify format (with data owner permission)
- Compare seed audience column names with training dataset definition

## Phase 2 — Remediate

MUST:
- Ensure seed audience has minimum 500+ records
- Fix schema alignment — column names and types must match training dataset
- Verify CSV format with proper delimiters and encoding (UTF-8)

SHOULD:
- Remove duplicate records from seed audience
- Validate no null or empty required fields
- Ensure seed audience is representative of the target segment

MAY:
- Increase seed audience to 1000+ records for better lookalike quality
- Pre-process seed data to standardize formats before upload

## Common Issues

- symptoms: "Audience generation fails with seed audience validation error"
  diagnosis: "Seed audience has fewer than minimum required records."
  resolution: "Increase seed audience to 500+ records. Verify record count excludes header row."

- symptoms: "Schema mismatch error during audience generation"
  diagnosis: "Seed audience columns do not match training dataset schema."
  resolution: "Align seed audience CSV columns with training dataset definition. Check column names, order, and types."

## Safety Ratings

```
safety_ratings:
  - "s3 ls: GREEN — read-only listing of seed data files"
  - "s3api head-object: GREEN — read-only metadata check"
  - "get-audience-generation-job: GREEN — read-only job details"
  - "s3 cp (upload corrected seed): YELLOW — writes data to S3"
```

## Escalation Conditions

- Seed audience data quality issues persist after multiple correction attempts
- Seed audience data is provided by external collaboration member and cannot be modified
- Schema requirements are unclear or undocumented
- Seed audience contains PII that requires special handling

## Data Sensitivity

- `aws s3 ls s3://<seed-bucket>/`: MEDIUM — reveals file names and sizes
- `aws s3api head-object`: LOW — returns metadata only
- Seed audience content: HIGH — contains audience segment data from collaboration members
- `aws cleanroomsml get-audience-generation-job`: MEDIUM — reveals job configuration

## Prohibited Actions

- NEVER download or inspect seed audience data content without explicit data owner permission
- NEVER modify seed audience data belonging to another collaboration member
- NEVER share seed audience S3 paths or content outside the collaboration
- NEVER use seed audience data for purposes outside the collaboration agreement

## Phase 3 — Rollback

1. If incorrect seed data was uploaded, delete it: `aws s3 rm s3://<bucket>/<key>` (with data owner approval)
2. Upload corrected seed audience to the same S3 location
3. Verify the corrected seed audience passes validation before retrying audience generation
4. Document the data quality issue for the collaboration team
5. Ensure no audience generation jobs ran with the incorrect seed data

## Output Format

```yaml
root_cause: "seed_audience_error — <specific_cause>"
evidence:
  - type: seed_data_error
    content: "<error details from job or S3 inspection>"
severity: HIGH
mitigation:
  immediate: "Fix seed audience data and retry"
  long_term: "Implement seed audience validation pipeline"
```

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
