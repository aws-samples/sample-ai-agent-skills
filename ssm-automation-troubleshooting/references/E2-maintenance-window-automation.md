---
title: "E2 — Maintenance Window Automation"
description: "Diagnose maintenance window automation task failures including scheduling, service roles, and task execution"
status: active
severity: HIGH
triggers: ["maintenance window failed", "window task error", "scheduled automation not running", "maintenance window task timeout"]
owner: devops-agent
objective: "Resolve maintenance window automation task execution failures"
context: "Maintenance windows schedule automation tasks to run during defined time windows. Tasks use the maintenance window service role, not the automation document's AssumeRole. Windows have their own scheduling, targeting, and error handling."
---

## Phase 1 — Triage

MUST:
- List maintenance windows: `aws ssm describe-maintenance-windows --filters "Key=Enabled,Values=true"`
- Get window execution history: `aws ssm describe-maintenance-window-executions --window-id <window-id>`
- Get task execution details: `aws ssm describe-maintenance-window-execution-task-invocations --window-execution-id <execution-id> --task-id <task-id>`

SHOULD:
- Check window tasks: `aws ssm describe-maintenance-window-tasks --window-id <window-id>`
- Verify the maintenance window service role: `aws iam get-role --role-name <service-role>`
- Check window schedule and next execution time

MAY:
- Review CloudTrail for maintenance window execution events
- Check if the window overlaps with Change Calendar closures

## Phase 2 — Remediate

MUST:
- Fix the maintenance window service role permissions
- Correct task configuration (document name, parameters, targets)
- Ensure the window schedule is correct and the window is enabled

SHOULD:
- Verify the service role has permissions for both SSM and target service actions
- Check that task targets are valid and accessible during the window
- Configure task error thresholds (MaxConcurrency, MaxErrors)

MAY:
- Set up SNS notifications for window execution results
- Implement pre-window validation tasks

## Safety Ratings

```
safety_ratings:
  - "describe-maintenance-windows: GREEN — read-only window listing"
  - "describe-maintenance-window-executions: GREEN — read-only execution history"
  - "update-maintenance-window-task: YELLOW — modifies scheduled task configuration"
  - "deregister-task-from-maintenance-window: YELLOW — removes scheduled task"
```

## Escalation Conditions

- Maintenance window automation fails on production patching schedule
- Service role permissions cannot be determined from error messages
- Window execution affects critical infrastructure during business hours
- Multiple consecutive window executions fail

## Data Sensitivity

- `aws ssm describe-maintenance-windows`: LOW — returns window metadata
- `aws ssm describe-maintenance-window-executions`: MEDIUM — contains execution results
- `aws ssm describe-maintenance-window-tasks`: MEDIUM — contains task parameters
- `aws iam get-role`: LOW — returns role metadata and trust policy

## Prohibited Actions

- NEVER disable a production maintenance window without scheduling an alternative
- NEVER modify maintenance window service roles without understanding all tasks that use them
- NEVER change window schedules during active execution windows
- NEVER remove error thresholds from maintenance window tasks

## Phase 3 — Rollback

1. If a task was misconfigured, revert to previous configuration
2. If the service role was modified, restore previous permissions
3. If the window schedule was changed, restore the original schedule
4. Verify the next window execution will use correct configuration
5. Manually run missed automation tasks if the window was missed

## Common Issues

- symptoms: "Maintenance window task fails with AccessDenied"
  diagnosis: "The maintenance window service role lacks required permissions."
  resolution: "Update the service role with permissions for all actions the automation document performs. Note: this is the window role, not the document's AssumeRole."

- symptoms: "Maintenance window executes but no targets are found"
  diagnosis: "Task targets do not match any instances or resources during the window."
  resolution: "Verify target tags, instance IDs, or resource groups. Ensure instances are running during the window."

- symptoms: "Maintenance window does not execute at scheduled time"
  diagnosis: "The window is disabled, the schedule expression is incorrect, or the window has no registered tasks."
  resolution: "Verify window is enabled. Check schedule expression (cron or rate). Ensure tasks are registered."

## Output Format

```yaml
root_cause: "maintenance_window — <specific_cause>"
evidence:
  - type: window_execution_error
    content: "<task execution failure details>"
severity: HIGH
mitigation:
  immediate: "Fix window task configuration"
  long_term: "Implement window execution monitoring"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
