---
title: "G2 — Incremental Transfer Issues"
description: "Diagnose incremental transfer detection and behavior problems"
status: active
severity: MEDIUM
triggers:
  - "incremental transfer"
  - "files not detected as changed"
  - "full transfer instead of incremental"
  - "deleted files not removed"
owner: devops-agent
objective: "Identify and resolve incremental transfer behavior issues"
context: "DataSync performs incremental transfers by default on subsequent executions, only transferring changed files. Change detection uses file metadata (size, timestamps). The PreserveDeletedFiles option controls whether deleted source files are removed from destination. The PREPARING phase scans source and destination to detect changes."
---

## Phase 1 — Triage

MUST:
- Check task transfer options: `aws datasync describe-task --task-arn <task-arn> --query 'Options.{OverwriteMode:OverwriteMode,PreserveDeletedFiles:PreserveDeletedFiles,TransferMode:TransferMode}'`
- Check execution results for files transferred vs files scanned: `aws datasync describe-task-execution --task-execution-arn <execution-arn>`
- Verify TransferMode is set to CHANGED (default) not ALL
- Check if source file timestamps are being modified between runs

SHOULD:
- Compare file counts between source and destination
- Check if PreserveDeletedFiles is PRESERVE (keeps deleted files) or REMOVE
- Review PREPARING phase duration (indicates scanning scope)

MAY:
- Check if OverwriteMode is ALWAYS (overwrites all) or NEVER
- Verify file system clock synchronization between source and destination

## Phase 2 — Remediate

MUST:
- For incremental transfers, ensure TransferMode=CHANGED (default)
- To remove deleted files from destination: set PreserveDeletedFiles=REMOVE
- To force full transfer: set TransferMode=ALL (use sparingly)

SHOULD:
- Update task options: `aws datasync update-task --task-arn <task-arn> --options TransferMode=CHANGED,PreserveDeletedFiles=REMOVE`
- Ensure source file timestamps accurately reflect modifications

MAY:
- Use OverwriteMode=ALWAYS if timestamp-based detection is unreliable
- Implement source-side change tracking for large datasets

## Common Issues

- symptoms: "All files transferred every time instead of only changes"
  diagnosis: "TransferMode set to ALL, or source timestamps changing on every access."
  resolution: "Set TransferMode=CHANGED. Check if source filesystem updates atime on read."

- symptoms: "Deleted files still present at destination"
  diagnosis: "PreserveDeletedFiles is set to PRESERVE (default)."
  resolution: "Set PreserveDeletedFiles=REMOVE to delete files at destination when removed from source."

- symptoms: "Changed files not detected"
  diagnosis: "File content changed but size and timestamps unchanged."
  resolution: "DataSync uses metadata for change detection. Touch files to update timestamps, or use TransferMode=ALL."

## Output Format

```yaml
root_cause: "incremental_transfer — <specific_cause>"
evidence:
  - type: transfer_options
    content: "<TransferMode, PreserveDeletedFiles, OverwriteMode>"
  - type: execution_results
    content: "<files scanned vs files transferred>"
  - type: file_metadata
    content: "<source file timestamps and sizes>"
severity: MEDIUM
mitigation:
  immediate: "Configure correct transfer mode and deleted file handling"
  long_term: "Document incremental transfer behavior and monitor execution results"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Transfer mode affects which files are copied and deleted |
| Blast Radius | HIGH — PreserveDeletedFiles=REMOVE deletes files at destination |
| Rollback Complexity | MEDIUM — Deleted files at destination cannot be recovered without backup |
| Compliance Scope | MEDIUM — File deletion behavior must align with data retention policies |

## Escalation Conditions

- Escalate to data owners before enabling PreserveDeletedFiles=REMOVE
- Escalate to compliance if file deletion behavior conflicts with retention policies
- Escalate to operations if TransferMode=ALL causes unnecessary full transfers

## Data Sensitivity

- **Transfer mode settings** control data lifecycle at destination
- **Deleted file lists** may contain sensitive file paths

## PROHIBITED Actions

- Do NOT set PreserveDeletedFiles=REMOVE without data owner approval
- Do NOT set TransferMode=ALL for routine transfers — it wastes bandwidth and time
- Do NOT change incremental transfer settings on production tasks without testing
- Do NOT assume deleted files at destination can be recovered

## Phase 3 — Rollback

- Restore previous transfer options: `aws datasync update-task --task-arn <arn> --options TransferMode=CHANGED,PreserveDeletedFiles=PRESERVE`
- If files were incorrectly deleted at destination, restore from backup
- Re-run the task with corrected settings to restore proper state

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
