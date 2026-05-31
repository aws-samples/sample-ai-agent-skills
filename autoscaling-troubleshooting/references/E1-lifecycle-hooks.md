---
title: "E1 — Lifecycle Hook Issues"
description: "Diagnose lifecycle hook timeouts and configuration problems"
status: active
severity: MEDIUM
triggers:
  - "lifecycle hook timeout"
  - "instance stuck in pending"
  - "hook not completing"
  - "lifecycle action failed"
  - "instance not transitioning"
owner: devops-agent
objective: "Identify and resolve lifecycle hook issues"
context: "Lifecycle hooks pause instance transitions to allow custom actions (configuration, testing, etc.). Issues include: hook timeout too short, notification target not receiving events, completion action not called, or default result causing unexpected behavior."
---

## Phase 1 — Triage

MUST:
- Check lifecycle hooks: `aws autoscaling describe-lifecycle-hooks --auto-scaling-group-name <asg-name>`
- Check for instances stuck in Pending:Wait or Terminating:Wait: `aws autoscaling describe-auto-scaling-instances --query 'AutoScalingInstances[?AutoScalingGroupName==`<asg-name>`].{Id:InstanceId,State:LifecycleState}'`
- Check hook configuration: timeout, default result, notification target
- Check scaling activities for hook-related messages: `aws autoscaling describe-scaling-activities --auto-scaling-group-name <asg-name>`

SHOULD:
- Verify notification target (SNS/SQS/EventBridge) is receiving events
- Check if the automation completing the hook is functioning
- Verify IAM role for notification delivery

MAY:
- Check CloudWatch Logs for hook automation errors
- Review EventBridge rules for lifecycle events

## Phase 2 — Remediate

MUST:
- If stuck instances: complete the lifecycle action manually: `aws autoscaling complete-lifecycle-action --lifecycle-hook-name <hook> --auto-scaling-group-name <asg-name> --instance-id <id> --lifecycle-action-result CONTINUE`
- If timeout too short: increase heartbeat timeout
- If default result wrong: update to appropriate action (CONTINUE or ABANDON)

SHOULD:
- Extend heartbeat if needed: `aws autoscaling record-lifecycle-action-heartbeat --lifecycle-hook-name <hook> --auto-scaling-group-name <asg-name> --instance-id <id>`
- Update hook timeout: `aws autoscaling put-lifecycle-hook --lifecycle-hook-name <hook> --auto-scaling-group-name <asg-name> --heartbeat-timeout 7200`

MAY:
- Implement robust error handling in hook automation
- Set up monitoring for hook completion times

## Output Format

```yaml
root_cause: "lifecycle_hook_issue — <specific_cause>"
evidence:
  - type: hook_config
    content: "<hook configuration>"
  - type: stuck_instances
    content: "<instances in Wait state>"
severity: MEDIUM
mitigation:
  immediate: "Complete stuck lifecycle actions or increase timeout"
  long_term: "Implement reliable hook automation with monitoring"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Complete lifecycle action manually | YELLOW | Transitions instance — verify readiness first |
| Increase heartbeat timeout | GREEN | Extends wait time — non-destructive |
| Update default result (CONTINUE/ABANDON) | YELLOW | Changes behavior on timeout — affects instance lifecycle |
| Extend heartbeat | GREEN | Extends current wait — non-destructive |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- Completing lifecycle actions transitions instances to active state
- Default result changes affect all future lifecycle transitions

## Data Sensitivity

- Lifecycle hook configuration reveals instance initialization processes
- Notification targets may contain sensitive operational data
- Hook automation logs may contain initialization details

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER complete lifecycle actions with CONTINUE for instances that failed initialization
- NEVER set default result to CONTINUE without verifying initialization automation works

## Phase 3 — Rollback

- If lifecycle action was completed incorrectly: terminate the instance and let ASG replace it
- If timeout was changed: restore original timeout value
- If default result was changed: restore original default result
- Document all lifecycle hook changes for operational audit trail

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
