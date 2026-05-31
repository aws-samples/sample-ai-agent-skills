---
title: "C3 — Warm Pool"
description: "Diagnose warm pool configuration and transition issues"
status: active
severity: MEDIUM
triggers:
  - "warm pool not working"
  - "warm pool instances not launching"
  - "warm pool transition failed"
  - "pre-warmed instances"
  - "warm pool stuck"
owner: devops-agent
objective: "Identify and resolve warm pool issues"
context: "Warm pool pre-initializes instances for faster scale-out. Issues include: pool not filling, instances stuck in transition, lifecycle hooks blocking, incorrect pool state configuration, or MaxGroupPreparedCapacity misconfigured."
---

## Phase 1 — Triage

MUST:
- Check warm pool configuration: `aws autoscaling describe-warm-pool --auto-scaling-group-name <asg-name>`
- Check warm pool instance states: `aws autoscaling describe-warm-pool --auto-scaling-group-name <asg-name> --query 'Instances[*].{Id:InstanceId,State:LifecycleState,Health:HealthStatus}'`
- Check ASG capacity vs warm pool: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> --query 'AutoScalingGroups[*].{Desired:DesiredCapacity,Max:MaxSize,WarmPoolSize:WarmPoolSize}'`
- Check lifecycle hooks for warm pool: `aws autoscaling describe-lifecycle-hooks --auto-scaling-group-name <asg-name>`

SHOULD:
- Verify MaxGroupPreparedCapacity is set appropriately
- Check if warm pool instances are in expected state (Stopped/Running/Hibernated)
- Review scaling activities for warm pool transitions

MAY:
- Check if instance type supports hibernation (if using Hibernated state)
- Verify warm pool reuse policy

## Phase 2 — Remediate

MUST:
- If pool not filling: check launch failures and capacity
- If transitions stuck: check lifecycle hooks and complete them
- Complete stuck lifecycle action: `aws autoscaling complete-lifecycle-action --lifecycle-hook-name <hook> --auto-scaling-group-name <asg-name> --instance-id <id> --lifecycle-action-result CONTINUE`

SHOULD:
- Configure warm pool: `aws autoscaling put-warm-pool --auto-scaling-group-name <asg-name> --pool-state Stopped --max-group-prepared-capacity <n>`
- Use Running state for fastest scale-out (higher cost)

MAY:
- Implement lifecycle hooks for warm pool initialization
- Monitor warm pool utilization metrics

## Output Format

```yaml
root_cause: "warm_pool_issue — <specific_cause>"
evidence:
  - type: warm_pool_config
    content: "<pool configuration>"
  - type: instance_states
    content: "<warm pool instance states>"
severity: MEDIUM
mitigation:
  immediate: "Fix warm pool configuration or lifecycle hooks"
  long_term: "Optimize warm pool size and state for workload patterns"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Complete stuck lifecycle action | YELLOW | Transitions instance — may affect warm pool state |
| Configure warm pool | YELLOW | Adds pre-initialized instances — increases cost |
| Use Running state for fastest scale-out | YELLOW | Higher cost — instances running but not serving |
| Implement lifecycle hooks for warm pool | YELLOW | Adds initialization steps — may delay transitions |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- Warm pool configuration affects scale-out speed and cost
- Lifecycle hook changes affect instance initialization process

## Data Sensitivity

- Warm pool configuration reveals capacity planning and scaling strategy
- Instance states reveal infrastructure readiness posture
- Lifecycle hook configuration may reveal initialization processes

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER complete lifecycle actions with ABANDON without understanding the impact
- NEVER delete warm pool during active scaling events

## Phase 3 — Rollback

- If warm pool was configured incorrectly: update configuration or delete with `aws autoscaling delete-warm-pool`
- If lifecycle action was completed incorrectly: the instance may need to be terminated and replaced
- If warm pool state was changed: update pool state configuration
- Document all warm pool changes for operational audit trail

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
