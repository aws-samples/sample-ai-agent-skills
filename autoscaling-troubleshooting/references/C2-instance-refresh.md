---
title: "C2 — Instance Refresh"
description: "Diagnose instance refresh failures and stuck operations"
status: active
severity: MEDIUM
triggers:
  - "instance refresh failed"
  - "instance refresh stuck"
  - "refresh not completing"
  - "rolling update failed"
  - "refresh cancelled"
owner: devops-agent
objective: "Identify why instance refresh is failing and resolve the issue"
context: "Instance refresh replaces instances to match a new launch template or configuration. Failures occur due to: health check failures during replacement, MinHealthyPercentage too restrictive, insufficient capacity, lifecycle hook timeouts, or application deployment issues."
---

## Phase 1 — Triage

MUST:
- Check instance refresh status: `aws autoscaling describe-instance-refreshes --auto-scaling-group-name <asg-name> --query 'InstanceRefreshes[0].{Id:InstanceRefreshId,Status:Status,Progress:PercentageComplete,Reason:StatusReason}'`
- Check refresh preferences: `aws autoscaling describe-instance-refreshes --auto-scaling-group-name <asg-name> --query 'InstanceRefreshes[0].Preferences'`
- Check for failed instances during refresh: `aws autoscaling describe-scaling-activities --auto-scaling-group-name <asg-name> --max-items 20`
- Verify current ASG health: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name>`

SHOULD:
- Check if MinHealthyPercentage is achievable with current capacity
- Verify new launch template version works (test manual launch)
- Check lifecycle hooks that may block transitions

MAY:
- Review checkpoint configuration for large ASGs
- Check if SkipMatching is enabled to avoid unnecessary replacements

## Phase 2 — Remediate

MUST:
- If health check failures: fix application or health check config before retrying
- If stuck: cancel and retry with adjusted preferences
- Cancel stuck refresh: `aws autoscaling cancel-instance-refresh --auto-scaling-group-name <asg-name>`

SHOULD:
- Start refresh with appropriate settings: `aws autoscaling start-instance-refresh --auto-scaling-group-name <asg-name> --preferences '{"MinHealthyPercentage":90,"InstanceWarmup":300,"SkipMatching":true}'`
- Use rollback if available: `aws autoscaling rollback-instance-refresh --auto-scaling-group-name <asg-name>`

MAY:
- Use checkpoints for large ASGs to pause and verify
- Implement canary deployment before full refresh

## Output Format

```yaml
root_cause: "instance_refresh_failure — <specific_cause>"
evidence:
  - type: refresh_status
    content: "<refresh state and progress>"
  - type: failure_reason
    content: "<StatusReason>"
severity: MEDIUM
mitigation:
  immediate: "Cancel failed refresh and fix root cause"
  long_term: "Implement proper health checks and refresh preferences"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Cancel stuck refresh | YELLOW | Stops rolling update — may leave ASG in mixed state |
| Retry with adjusted preferences | YELLOW | Restarts rolling replacement of instances |
| Rollback instance refresh | RED | Reverts all instances — significant production impact |
| Use checkpoints for large ASGs | GREEN | Adds safety gates — non-destructive |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- Instance refresh replaces running production instances
- Rollback reverts all instances to previous configuration

## Data Sensitivity

- Instance refresh status reveals deployment progress and failures
- Refresh preferences reveal deployment strategy and risk tolerance
- Launch template versions reveal infrastructure configuration changes

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER start instance refresh without verified launch template
- NEVER set MinHealthyPercentage to 0 in production (allows all instances to be replaced simultaneously)

## Phase 3 — Rollback

- If refresh was cancelled: verify ASG is in stable state with healthy instances
- If refresh with new preferences failed: cancel and investigate root cause before retrying
- If rollback was triggered: monitor until all instances are reverted to previous configuration
- Document all instance refresh operations for operational audit trail

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
