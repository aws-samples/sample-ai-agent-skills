---
title: "D1 — Journal Export Failures"
description: "Diagnose journal export to S3 failures"
status: active
severity: HIGH
triggers: ["export failed", "S3 error", "export stuck"]
owner: devops-agent
objective: "Identify and resolve journal export failures"
context: "Export fails when: S3 permissions wrong, bucket doesn't exist, KMS key blocks write, or time range invalid."
---

## Phase 1 — Triage

MUST:
- Check export status: `aws qldb describe-journal-s3-export --name <ledger> --export-id <id>`
- Verify S3 bucket exists and IAM role has s3:PutObject
- Check time range is valid (start before end, within ledger lifetime)
- Verify KMS key permissions on S3 bucket

SHOULD:
- Check export output prefix in S3
- Verify IAM role trust policy allows qldb.amazonaws.com

## Phase 2 — Remediate

MUST:
- Fix S3 bucket permissions
- Fix IAM role trust policy
- Correct time range

SHOULD:
- Use dedicated S3 prefix for exports
- Monitor export job progress

## Output Format

```yaml
root_cause: "journal_export — <specific_cause>"
severity: HIGH
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — journal exports contain complete ledger history
- Service Disruption Risk: LOW — exports do not affect ledger operations
- Compliance Impact: HIGH — journal exports are critical for audit compliance

## Escalation Conditions
- Journal export fails with permission errors
- Export output corrupted or incomplete
- Export S3 bucket access issues
- Export takes longer than expected
- Export data contains sensitive information requiring redaction

## Data Sensitivity
- Journal export data — CONFIDENTIAL (complete ledger history)
- S3 output configurations — INTERNAL
- Export IAM role configurations — INTERNAL
- Export scheduling — INTERNAL
- Never expose journal export data to unauthorized parties

## Prohibited Actions
- Do NOT store journal exports in publicly accessible S3 buckets
- Do NOT disable encryption for journal export output
- Do NOT share journal exports with unauthorized parties
- Do NOT retain exports beyond approved retention periods
- Do NOT grant export roles access to unrelated S3 buckets
- Do NOT ignore export failures for compliance-required exports

## Phase 3 — Rollback
- Delete incorrectly exported data
- Restore S3 bucket configurations
- Revert export role permissions
- Re-run export with corrected configurations
- Verify export data integrity
- Document export issues and compliance impact

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
