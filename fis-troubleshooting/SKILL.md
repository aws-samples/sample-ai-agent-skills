---
name: fis-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Fault Injection
  Simulator (FIS) problems by analyzing experiment templates, target
  selection, actions, stop conditions, IAM roles, experiment execution,
  and following structured runbooks.
  Activate when: experiment template errors, target resolution failures,
  action execution problems, stop condition triggers, IAM permission
  issues, experiment failures, or the user says something is wrong
  with FIS without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with fis, ec2, ecs, rds, iam,
  cloudwatch, and cloudtrail permissions.
---

# AWS Fault Injection Service Diagnostics

## When to use

Any AWS FIS investigation — experiment template creation errors, target selection failures, action execution problems, stop condition configuration, IAM role issues, or experiment execution failures.

## Investigation workflow

### Step 1 — Collect and triage

```
aws fis list-experiment-templates --query 'experimentTemplates[*].{Id:id,Description:description}'
aws fis list-experiments --query 'experiments[*].{Id:id,State:state.status,Template:experimentTemplateId}'
aws fis list-target-account-configurations --experiment-template-id <template-id>
```

### Step 2 — Domain deep dive

```
aws fis get-experiment-template --id <template-id>
aws fis get-experiment --id <experiment-id>
aws iam get-role --role-name <fis-role-name>
aws cloudwatch describe-alarms --alarm-names <stop-condition-alarm>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=fis.amazonaws.com --max-results 20
aws fis list-actions --query 'actions[*].{Id:id,Description:description}'
```

Read `references/guardrails.md` before concluding on any FIS issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `fis list-experiment-templates` | List templates |
| `fis get-experiment-template` | Get template details |
| `fis start-experiment` | Start an experiment |
| `fis get-experiment` | Check experiment status |
| `fis stop-experiment` | Stop running experiment |
| `fis list-actions` | List available FIS actions |

## Gotchas: AWS Fault Injection Service

- FIS experiments cause REAL failures in your infrastructure. They are not simulations — they actually stop instances, inject latency, etc. Always use stop conditions.
- The FIS IAM role must have permissions to perform the actions AND to the target resources. A common mistake is granting FIS permissions but not resource-level permissions.
- Stop conditions use CloudWatch alarms. If the alarm transitions to ALARM state, FIS stops the experiment. The alarm must exist BEFORE the experiment starts.
- Target selection uses filters (tags, resource IDs, etc.) and selection mode (ALL, COUNT, PERCENT). If filters match no resources, the experiment fails.
- Some actions are irreversible (e.g., terminate instances). FIS does not restore resources after experiment completion.
- Experiment duration and action timing matter. Actions execute in parallel by default unless you define dependencies.
- Multi-account experiments require target account configurations and cross-account IAM roles.

## Anti-hallucination rules

1. Always cite specific experiment IDs, template configs, or API responses as evidence.
2. FIS causes REAL failures. Never describe it as a simulation that doesn't affect production.
3. Stop conditions require pre-existing CloudWatch alarms. Never assume auto-creation.
4. Some actions are irreversible. Always warn about destructive actions.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Templates | A1-A2 | Template creation, template validation |
| B — Targets | B1-B2 | Target selection, target resolution |
| C — Actions | C1-C3 | Action configuration, action execution, action dependencies |
| D — Stop Conditions | D1-D2 | Stop condition setup, stop condition triggers |
| E — IAM | E1-E2 | Role configuration, permission issues |
| F — Execution | F1-F2 | Experiment failures, experiment monitoring |
| Z — Catch-All | Z1 | General troubleshooting |
