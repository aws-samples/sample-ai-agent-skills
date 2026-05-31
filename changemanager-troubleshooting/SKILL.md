---
name: changemanager-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Systems Manager
  Change Manager problems by analyzing change templates, change requests,
  approval workflows, runbook execution, scheduling, and following
  structured runbooks.
  Activate when: change template errors, change request failures,
  approval workflow issues, runbook execution problems, scheduling
  conflicts, or the user says something is wrong with Change Manager.
compatibility: >
  Requires AWS CLI or SDK access with ssm, iam, cloudtrail, sns,
  and organizations permissions.
---

# AWS SSM Change Manager Diagnostics

## When to use

Any SSM Change Manager investigation — change template creation, change request submission, approval workflow issues, runbook execution failures, or scheduling problems.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ssm list-documents --filters "Key=DocumentType,Values=ChangeTemplate" --query 'DocumentIdentifiers[*].{Name:Name,Status:DocumentStatus}'
version: "1.0.0"
last_updated: "2025-04-12"
aws ssm list-change-requests-by-approval-status --filters "Key=ChangeRequestStatus,Values=Pending,Approved,Rejected"
aws ssm list-ops-item-events --filters "Key=OpsItemType,Values=/aws/changerequest"
```

### Step 2 — Domain deep dive

```
aws ssm describe-document --name <template-name>
aws ssm get-ops-item --ops-item-id <change-request-id>
aws ssm list-document-versions --name <template-name>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ssm.amazonaws.com --max-results 20
aws ssm describe-automation-executions --filters "Key=ExecutionId,Values=<execution-id>"
aws sns list-topics
```

Read `references/guardrails.md` before concluding on any Change Manager issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `ssm list-documents` (ChangeTemplate) | List change templates |
| `ssm describe-document` | Get template details |
| `ssm get-ops-item` | Get change request details |
| `ssm describe-automation-executions` | Check runbook execution |
| `ssm start-change-request-execution` | Submit change request |

## Gotchas: AWS SSM Change Manager

- Change Manager requires an SNS topic for notifications. Without it, approvers don't receive notification of pending requests.
- Approval workflows can have multiple levels. ALL approvers at each level must approve before proceeding to the next level.
- Change templates define the runbook, approval requirements, and notification settings. Templates must be approved before they can be used.
- Scheduled changes execute at the specified time only if all approvals are complete. Unapproved scheduled changes are NOT executed.
- Change Manager integrates with Organizations for multi-account changes. The management account or delegated admin configures organization-wide settings.
- Runbook execution failures don't automatically roll back changes. Rollback must be handled within the runbook logic.

## Anti-hallucination rules

1. Always cite specific change request IDs, template names, or API responses as evidence.
2. ALL approvers at each level must approve. Never claim majority approval is sufficient.
3. Unapproved scheduled changes don't execute. Never assume auto-approval.
4. Runbook failures don't auto-rollback. Never claim automatic rollback.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Templates | A1-A2 | Template creation, template approval |
| B — Requests | B1-B2 | Request submission, request status |
| C — Approvals | C1-C3 | Approval workflow, approver notification, approval timeout |
| D — Execution | D1-D2 | Runbook execution, execution failures |
| E — Scheduling | E1-E2 | Schedule configuration, schedule conflicts |
| Z — Catch-All | Z1 | General troubleshooting |
