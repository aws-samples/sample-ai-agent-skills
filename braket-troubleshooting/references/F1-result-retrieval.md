---
title: "F1 — Result Retrieval"
description: "Diagnose quantum task result retrieval and S3 output issues"
status: active
severity: MEDIUM
triggers: ["cannot get results", "results not found", "S3 output missing", "result retrieval error"]
owner: devops-agent
objective: "Resolve quantum task result retrieval failures"
context: "Braket stores task results in S3. Retrieval issues stem from S3 permissions, bucket configuration, or task status."
---

## Phase 1 — Triage

MUST:
- Check task status: `aws braket get-quantum-task --quantum-task-arn <arn>`
- Verify task is in COMPLETED status (results only available after completion)
- Check S3 output location: `aws s3 ls s3://<output-bucket>/tasks/<task-id>/`

SHOULD:
- Verify S3 bucket exists and caller has s3:GetObject permission
- Check if the output bucket has a bucket policy restricting access
- Verify the S3 output prefix matches the expected path

MAY:
- Check S3 bucket versioning for accidentally overwritten results
- Review S3 access logs for permission denied events

## Phase 2 — Remediate

MUST:
- Ensure task is COMPLETED before attempting result retrieval
- Fix S3 permissions to allow GetObject on the output bucket
- Verify the correct S3 path: `s3://<bucket>/tasks/<task-id>/results.json`

SHOULD:
- Use Braket SDK result() method which handles S3 retrieval automatically
- Verify S3 bucket region matches the Braket task region
- Check for S3 bucket lifecycle rules that may have deleted results

MAY:
- Set up S3 event notifications for result availability
- Implement result caching to avoid repeated S3 retrievals

## Safety Ratings

```
safety_ratings:
  - "get-quantum-task: GREEN — read-only task status"
  - "s3 ls: GREEN — read-only bucket listing"
  - "s3 cp (download results): GREEN — read-only data retrieval"
  - "s3 bucket policy modification: RED — affects access for all users"
```

## Escalation Conditions

- Results are missing from S3 for COMPLETED tasks
- S3 bucket was deleted or results were purged by lifecycle rules
- Cross-account S3 access issues that require bucket policy changes
- Results are corrupted or incomplete

## Data Sensitivity

- `aws braket get-quantum-task`: LOW — returns task metadata
- `aws s3 ls`: LOW — returns object keys
- Task results (results.json): MEDIUM — contains quantum computation outcomes
- S3 bucket policies: MEDIUM — reveals access control configuration

## Prohibited Actions

- NEVER modify S3 bucket policies without understanding the impact on other users
- NEVER delete S3 objects in the Braket output bucket
- NEVER assume results are available immediately after task submission
- NEVER grant public access to the Braket output bucket to fix permission issues

## Phase 3 — Rollback

1. If S3 permissions were modified, revert bucket policy to original
2. If results were accidentally deleted, check S3 versioning for recovery
3. Restore any IAM policy changes made during troubleshooting
4. Document the correct S3 path format for future reference
5. Verify other tasks' results are still accessible after any permission changes

## Common Issues

- symptoms: "SDK result() call returns 'task not completed' error"
  diagnosis: "Task is still in QUEUED or RUNNING status."
  resolution: "Wait for task to reach COMPLETED status. Use task.state() to poll status."

- symptoms: "S3 AccessDenied when retrieving results"
  diagnosis: "IAM role or user lacks s3:GetObject permission on the output bucket."
  resolution: "Add s3:GetObject permission for the Braket output bucket ARN to the IAM policy."

## Output Format

```yaml
root_cause: "result_retrieval — <specific_cause>"
evidence:
  - type: retrieval_error
    content: "<task status, S3 path, permission details>"
severity: MEDIUM
mitigation:
  immediate: "Fix S3 access or wait for task completion"
  long_term: "Implement automated result retrieval with proper error handling"
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
