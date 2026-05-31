---
name: ssm-automation-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Systems Manager
  Automation problems by analyzing automation execution failures,
  step errors, document creation, versioning, approval workflows,
  rate control, cross-account automation, multi-region execution,
  change calendar integration, maintenance windows, custom actions,
  output parameters, and service roles.
  Activate when: automation execution failures, step execution errors,
  document creation issues, approval workflow failures, rate control
  problems, cross-account automation errors, multi-region execution
  issues, change calendar blocks, maintenance window automation
  failures, custom action errors, output parameter issues, service
  role problems, or the user says something is wrong with SSM Automation.
compatibility: >
  Requires AWS CLI or SDK access with ssm, iam, sts, cloudtrail,
  cloudwatch, organizations, and lambda permissions.
---

# SSM Automation Diagnostics

## When to use

Any SSM Automation investigation — execution failures, step errors, document creation, versioning, approval workflows, rate control, cross-account automation, multi-region execution, change calendar integration, maintenance window automation, custom actions, output parameters, or service role configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ssm describe-automation-executions --filters Key=ExecutionStatus,Values=Failed --max-results 10
aws ssm list-documents --document-filter-list key=DocumentType,value=Automation --max-results 20
aws ssm describe-maintenance-windows --filters "Key=Enabled,Values=true"
```

### Step 2 — Domain deep dive

```
aws ssm get-automation-execution --automation-execution-id <execution-id>
aws ssm describe-automation-step-executions --automation-execution-id <execution-id>
aws ssm describe-document --name <document-name> --document-version '$LATEST'
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ssm.amazonaws.com --max-results 20
aws ssm get-calendar-state --calendar-names <calendar-name>
aws iam get-role --role-name <automation-role-name>
```

Read `references/guardrails.md` before concluding on any SSM Automation issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `ssm describe-automation-executions` | List automation executions with status |
| `ssm get-automation-execution` | Get execution details and step outputs |
| `ssm describe-automation-step-executions` | Get individual step details |
| `ssm describe-document` | Get document schema and parameters |
| `ssm list-document-versions` | List document versions |
| `ssm get-calendar-state` | Check change calendar open/closed state |
| `ssm describe-maintenance-windows` | List maintenance windows |

## Gotchas: SSM Automation

- Automation documents have TWO execution modes: Simple (sequential steps) and Rate Control (concurrent targets). Rate control requires MaxConcurrency and MaxErrors parameters.
- Cross-account automation requires a management account or delegated admin in Organizations, plus target account IAM roles with proper trust policies.
- Change Calendar integration blocks automation execution when the calendar is CLOSED. Automations fail with CalendarStateIsClosedException, not a permissions error.
- Approval steps have a configurable timeout (default 7 days). If no approver acts, the execution times out — it does NOT auto-approve or auto-reject.
- Output parameters from one step are referenced as `{{stepName.outputKey}}` in subsequent steps. Incorrect references cause silent failures with empty values, not errors.
- Service roles for automation need both ssm:* permissions AND permissions for the actions the document performs (ec2, s3, lambda, etc.).
- Document versioning uses $DEFAULT and $LATEST. Executions use $DEFAULT unless explicitly specified. Updating a document does NOT change $DEFAULT automatically.

## Anti-hallucination rules

1. Always cite specific execution IDs, step names, or API responses as evidence.
2. Simple execution vs Rate Control execution have different failure modes. Never conflate them.
3. Cross-account automation requires Organizations setup. Never suggest it works with standalone accounts.
4. Change Calendar blocks executions, it does NOT modify them. Never claim calendars alter automation behavior.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Execution | A1-A2 | Automation execution failures, step execution errors |
| B — Documents | B1-B2 | Document creation issues, document versioning |
| C — Workflows | C1-C2 | Approval workflow failures, rate control issues |
| D — Multi-Scope | D1-D2 | Cross-account automation, multi-region execution |
| E — Scheduling | E1-E2 | Change calendar integration, maintenance window automation |
| F — Advanced | F1-F2 | Custom action errors, output parameter issues |
| G — IAM | G1 | Automation service role |
| Z — Catch-All | Z1 | General troubleshooting |
