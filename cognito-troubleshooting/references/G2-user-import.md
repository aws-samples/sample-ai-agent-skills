---
title: "G2 — User Import Issues"
description: "Diagnose bulk user import failures via CSV"
status: active
severity: MEDIUM
triggers:
  - "user import"
  - "CSV import"
  - "bulk import"
  - "import job"
  - "CreateUserImportJob"
owner: devops-agent
objective: "Identify and fix user import job failures"
context: "Cognito supports bulk user import via CSV files. Import jobs require a specific CSV format, an IAM role with CloudWatch Logs permissions, and correct attribute mapping. Users are imported with RESET_REQUIRED status."
---

## Phase 1 — Triage

MUST:
- Check import job status: `aws cognito-idp describe-user-import-job --user-pool-id <pool-id> --job-id <job-id>`
- Check CloudWatch Logs for import errors (Cognito writes import logs)
- Verify the CSV format matches the user pool schema
- Check the IAM role for the import job: `aws iam get-role --role-name <import-role>`

SHOULD:
- Verify the CSV header matches required and custom attributes
- Check for duplicate usernames in the CSV
- Verify phone numbers are in E.164 format and emails are valid

MAY:
- Download the import job results for detailed error analysis
- Check if the user pool is approaching user limits

## Phase 2 — Remediate

MUST:
- Fix CSV format errors (headers, attribute names, data format)
- Ensure the IAM role has logs:CreateLogGroup, logs:CreateLogStream, logs:PutLogEvents permissions
- Remove duplicate entries from the CSV

SHOULD:
- Validate the CSV locally before uploading
- Import in smaller batches for easier error tracking

MAY:
- Use the user migration trigger as an alternative for gradual migration

## Common Issues

- symptoms: "Import job stuck in Pending status"
  diagnosis: "IAM role doesn't have CloudWatch Logs permissions."
  resolution: "Add logs:CreateLogGroup, logs:CreateLogStream, logs:PutLogEvents to the role."

- symptoms: "Import job completes but many users failed"
  diagnosis: "CSV data doesn't match user pool schema or has format errors."
  resolution: "Check CloudWatch Logs for specific row errors. Fix CSV format."

- symptoms: "Imported users cannot sign in"
  diagnosis: "Imported users are in RESET_REQUIRED status and must set a new password."
  resolution: "Users must complete the password reset flow on first sign-in. This is expected behavior."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix CSV format | GREEN | Data preparation; no state change |
| Fix IAM role permissions | YELLOW | Permission change; recoverable |
| Re-run import job | YELLOW | Creates users; can be cleaned up |

## Escalation Conditions

- Production user pool bulk import operations
- IAM role changes for import jobs
- Large-scale user creation (>1000 users)

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-user-import-job` | MEDIUM | Job status and error counts |
| CloudWatch import logs | HIGH | May contain user attributes and PII |
| CSV file contents | HIGH | Contains user PII |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production
- NEVER suggest disabling advanced security features
- NEVER log or expose CSV contents containing user PII

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Import job execution | Delete imported users if needed via `admin-delete-user` |
| IAM role change | Revert IAM policy to previous version |

## Output Format

```yaml
root_cause: "user_import_failure — <specific_cause>"
evidence:
  - type: import_job_status
    content: "<job status and error counts>"
  - type: import_logs
    content: "<CloudWatch import error logs>"
severity: MEDIUM
mitigation:
  immediate: "Fix CSV format and re-run import"
  long_term: "Automate CSV validation before import"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-user-pool"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-users"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "admin-get-user"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling MFA for troubleshooting"
  - "NEVER expose user PII in investigation output"
  - "NEVER suggest removing password policies"
