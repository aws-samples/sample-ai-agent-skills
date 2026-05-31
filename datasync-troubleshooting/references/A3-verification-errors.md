---
title: "A3 — Verification Errors"
description: "Diagnose verification mismatches after DataSync transfer"
status: active
severity: MEDIUM
triggers:
  - "verification failed"
  - "verification mismatch"
  - "data integrity error"
  - "files differ after transfer"
owner: devops-agent
objective: "Identify why verification is reporting mismatches and determine if data integrity is compromised"
context: "Verification compares source and destination after transfer. Mismatches can occur due to source changes during transfer, metadata differences between location types, timestamp precision differences, or actual transfer errors. Verification failures do not necessarily mean the transfer failed."
---

## Phase 1 — Triage

MUST:
- Check task execution result for verification details: `aws datasync describe-task-execution --task-execution-arn <task-execution-arn>`
- Check verification mode configured on the task: `aws datasync describe-task --task-arn <task-arn> --query 'Options.VerifyMode'`
- Review CloudWatch Logs for specific file mismatches: `aws logs filter-log-events --log-group-name /aws/datasync --filter-pattern "Verification"`
- Check if source files changed during transfer

SHOULD:
- Compare file counts at source and destination
- Check metadata preservation settings in task options
- Verify timestamp handling between source and destination location types

MAY:
- Run a manual comparison of specific mismatched files
- Check if antivirus or backup software modified files during transfer

## Phase 2 — Remediate

MUST:
- Determine if mismatches are due to source changes or actual transfer errors
- Re-run the task if actual transfer errors occurred
- Document which files had verification mismatches

SHOULD:
- Use POINT_IN_TIME_CONSISTENT verification for critical data: `aws datasync update-task --task-arn <task-arn> --options VerifyMode=POINT_IN_TIME_CONSISTENT`
- Quiesce the source during transfer if source changes cause mismatches

MAY:
- Switch to NONE verification for non-critical bulk transfers to improve speed
- Implement pre/post transfer checksums outside of DataSync

## Common Issues

- symptoms: "Verification reports mismatches but files look correct"
  diagnosis: "Source files changed during transfer (POINT_IN_TIME_CONSISTENT mode)."
  resolution: "Quiesce source during transfer or use ONLY_FILES_TRANSFERRED mode."

- symptoms: "Metadata mismatches between NFS source and S3 destination"
  diagnosis: "S3 stores POSIX metadata differently than native filesystems."
  resolution: "Expected behavior. S3 stores metadata as object metadata. Verify data content is correct."

- symptoms: "Timestamp mismatches"
  diagnosis: "Different timestamp precision between source and destination."
  resolution: "Check PreserveTimestamps option. Some location types have different precision."

## Output Format

```yaml
root_cause: "verification_error — <specific_cause>"
evidence:
  - type: verification_mode
    content: "<configured verification mode>"
  - type: mismatch_details
    content: "<files and metadata that mismatched>"
  - type: source_changes
    content: "<whether source changed during transfer>"
severity: MEDIUM
mitigation:
  immediate: "Determine if mismatches are benign or indicate data loss"
  long_term: "Configure appropriate verification mode and quiesce source for critical transfers"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Verification errors may indicate data integrity issues |
| Blast Radius | LOW — Verification is read-only comparison |
| Rollback Complexity | LOW — Re-run task to fix transfer errors |
| Compliance Scope | HIGH — Data integrity verification is a compliance requirement for many workloads |

## Escalation Conditions

- Escalate IMMEDIATELY to data owners if verification indicates actual data loss or corruption
- Escalate to compliance if verification failures affect regulated data transfers
- Escalate to operations if source changes during transfer cannot be prevented

## Data Sensitivity

- **Verification results** may contain file paths — restrict access to authorized personnel
- **Mismatch details** in CloudWatch Logs may reveal file names and metadata

## PROHIBITED Actions

- Do NOT dismiss verification mismatches without investigation
- Do NOT disable verification for compliance-required data transfers
- Do NOT modify source data during active transfers without understanding the impact

## Phase 3 — Rollback

- If actual data corruption is confirmed, re-run the task with POINT_IN_TIME_CONSISTENT verification
- Quiesce the source before re-running to prevent source changes during transfer
- Verify data integrity at destination after successful re-run
- Document verification results for compliance audit trail

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
