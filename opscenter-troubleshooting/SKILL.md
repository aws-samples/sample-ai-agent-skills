---
name: opscenter-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Systems Manager
  OpsCenter problems by analyzing OpsItem creation, automation runbooks,
  related resources, deduplication, EventBridge integration, notifications,
  and following structured runbooks.
  Activate when: OpsItem creation failures, automation runbook errors,
  resource association issues, deduplication problems, EventBridge rule
  failures, notification delivery issues, or the user says something
  is wrong with OpsCenter.
compatibility: >
  Requires AWS CLI or SDK access with ssm, iam, events, sns,
  cloudtrail, and cloudwatch permissions.
---

# AWS SSM OpsCenter Diagnostics

## When to use

Any SSM OpsCenter investigation — OpsItem creation, automation runbook execution, related resource management, deduplication configuration, EventBridge integration, or notification delivery.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ssm describe-ops-items --ops-item-filters "Key=Status,Values=Open,Operator=Equal" --query 'OpsItemSummaries[*].{Id:OpsItemId,Title:Title,Status:Status,Severity:Severity,Source:Source}'
aws ssm get-service-setting --setting-id /ssm/opsCenter
aws events list-rules --name-prefix "SSM"
```

### Step 2 — Domain deep dive

```
aws ssm get-ops-item --ops-item-id <id>
aws ssm describe-automation-executions --filters "Key=ExecutionId,Values=<id>"
aws ssm list-ops-item-related-items --ops-item-id <id>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ssm.amazonaws.com --max-results 20
aws sns list-topics
aws events describe-rule --name <rule-name>
```

Read `references/guardrails.md` before concluding on any OpsCenter issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `ssm describe-ops-items` | List OpsItems with filters |
| `ssm get-ops-item` | Get OpsItem details |
| `ssm create-ops-item` | Create new OpsItem |
| `ssm update-ops-item` | Update OpsItem status/details |
| `ssm describe-automation-executions` | Check runbook execution |
| `ssm list-ops-item-related-items` | List related resources |

## Gotchas: AWS SSM OpsCenter

- OpsCenter is for operational issues, NOT incident management. It tracks and resolves operational issues using OpsItems.
- OpsItems can be created manually, by EventBridge rules, or by AWS services (Config, CloudWatch, etc.). Each source has different deduplication behavior.
- Automation runbooks associated with OpsItems require an IAM role with permissions for the actions in the runbook.
- Related resources are references (ARNs) attached to OpsItems. They are NOT live links — resource state changes don't auto-update the OpsItem.
- Deduplication uses an operational data hash. Duplicate OpsItems with the same hash are merged, not created as separate items.
- OpsCenter is regional. OpsItems in us-east-1 are not visible in eu-west-1. Use Explorer for cross-region aggregation.

## Anti-hallucination rules

1. Always cite specific OpsItem IDs, statuses, or API responses as evidence.
2. OpsCenter is NOT incident management. Never conflate with Incident Manager.
3. Related resources are static references. Never claim they auto-update.
4. Deduplication uses hash-based matching. Never assume content-based dedup.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — OpsItems | A1-A2 | Creation, management |
| B — Automation | B1-B2 | Runbooks, execution |
| C — Resources | C1-C2 | Related resources, association |
| D — Deduplication | D1-D2 | Dedup behavior, configuration |
| E — EventBridge | E1-E2 | Integration, event rules |
| F — Notifications | F1 | Notification delivery |
| Z — Catch-All | Z1 | General troubleshooting |
