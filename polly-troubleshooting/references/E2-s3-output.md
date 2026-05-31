---
title: "E2 — S3 Output Issues"
description: "Diagnose async task S3 output problems"
status: active
severity: MEDIUM
triggers: ["output not in S3", "S3 access denied", "wrong bucket"]
owner: devops-agent
objective: "Identify and resolve S3 output issues for async tasks"
context: "S3 output issues occur when: bucket doesn't exist, permissions wrong, KMS encryption blocks write, or prefix incorrect."
---

## Phase 1 — Triage

MUST:
- Verify S3 bucket exists: `aws s3api head-bucket --bucket <name>`
- Check bucket policy allows Polly to write
- Verify output key prefix in task configuration
- Check task status for S3-related errors

SHOULD:
- Check KMS key permissions if bucket uses encryption
- Verify bucket is in same region as Polly
- Check if output file exists: `aws s3 ls s3://<bucket>/<prefix>/`

## Phase 2 — Remediate

MUST:
- Create bucket in correct region
- Add bucket policy allowing Polly s3:PutObject
- Fix KMS key policy if using encryption

SHOULD:
- Use dedicated prefix for Polly output
- Monitor S3 for output files

## Output Format

```yaml
root_cause: "s3_output — <specific_cause>"
evidence:
  - type: s3_config
    content: "<bucket and permissions>"
severity: MEDIUM
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — S3 output may contain synthesized sensitive content
- Service Disruption Risk: LOW — S3 issues affect output storage, not synthesis
- Compliance Impact: MEDIUM — audio output storage must meet data handling requirements

## Escalation Conditions
- S3 output delivery fails
- S3 bucket permissions prevent output storage
- Output files corrupted or incomplete
- S3 bucket encryption not configured
- Output storage costs exceed expectations

## Data Sensitivity
- Audio output files — CONFIDENTIAL (may contain sensitive content)
- S3 bucket configurations — INTERNAL
- IAM role configurations — INTERNAL
- Output file metadata — INTERNAL
- Never expose audio output containing sensitive content

## Prohibited Actions
- Do NOT store audio output in publicly accessible S3 buckets
- Do NOT disable S3 encryption for audio output
- Do NOT grant broad S3 access to synthesis roles
- Do NOT retain audio output beyond approved retention periods
- Do NOT ignore S3 delivery failures
- Do NOT share S3 output paths externally

## Phase 3 — Rollback
- Restore S3 bucket configurations
- Revert IAM role permission changes
- Re-enable S3 encryption if disabled
- Verify output delivery after changes
- Delete exposed audio output if security incident
- Document S3 output issues and storage configurations

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
