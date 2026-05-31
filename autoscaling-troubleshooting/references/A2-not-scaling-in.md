---
title: "A2 — Not Scaling In"
description: "Diagnose why an Auto Scaling group is not removing instances when load decreases"
status: active
severity: MEDIUM
triggers:
  - "not scaling in"
  - "too many instances"
  - "instances not terminating"
  - "over-provisioned"
  - "scale-in not working"
owner: devops-agent
objective: "Identify why the ASG is not scaling in and resolve the issue"
context: "Auto Scaling groups may fail to scale in due to: scale-in protection enabled, DisableScaleIn on target tracking, minimum capacity reached, cooldown active, suspended processes, or instances in standby."
---

## Phase 1 — Triage

MUST:
- Check ASG min/desired: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> --query 'AutoScalingGroups[*].{Min:MinSize,Desired:DesiredCapacity,Count:length(Instances)}'`
- Check if DisableScaleIn is set on target tracking policies: `aws autoscaling describe-policies --auto-scaling-group-name <asg-name> --query 'ScalingPolicies[?PolicyType==`TargetTrackingScaling`].TargetTrackingConfiguration.DisableScaleIn'`
- Check for scale-in protected instances: `aws autoscaling describe-auto-scaling-instances --query 'AutoScalingInstances[?AutoScalingGroupName==`<asg-name>`].{Id:InstanceId,Protected:ProtectedFromScaleIn}'`
- Check suspended processes: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> --query 'AutoScalingGroups[*].SuspendedProcesses'`

SHOULD:
- Check for instances in standby: `aws autoscaling describe-auto-scaling-instances --query 'AutoScalingInstances[?AutoScalingGroupName==`<asg-name>` && LifecycleState==`Standby`]'`
- Verify scale-in alarm is in ALARM state
- Check recent scaling activities for scale-in failures

MAY:
- Review termination policy configuration
- Check for scheduled actions preventing scale-in

## Phase 2 — Remediate

MUST:
- If DisableScaleIn is true: evaluate if intentional, otherwise update policy
- If scale-in protection on all instances: remove protection from appropriate instances
- If min capacity too high: reduce MinSize
- If processes suspended: resume Terminate process

SHOULD:
- Remove scale-in protection: `aws autoscaling set-instance-protection --auto-scaling-group-name <asg-name> --instance-ids <id> --no-protected-from-scale-in`
- Enable scale-in on target tracking: update policy with DisableScaleIn=false

MAY:
- Implement scheduled scaling to reduce capacity during off-hours

## Output Format

```yaml
root_cause: "not_scaling_in — <specific_cause>"
evidence:
  - type: scale_in_protection
    content: "<protected instances>"
  - type: policy_config
    content: "<DisableScaleIn setting>"
severity: MEDIUM
mitigation:
  immediate: "Remove the specific scale-in blocker"
  long_term: "Review scale-in protection and policy settings regularly"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Evaluate and update DisableScaleIn | YELLOW | Changes scaling behavior — may cause capacity reduction |
| Remove scale-in protection from instances | YELLOW | Allows instances to be terminated |
| Reduce MinSize | YELLOW | May trigger immediate instance termination |
| Resume Terminate process | YELLOW | Re-enables termination — may trigger immediate actions |
| Implement scheduled scaling | YELLOW | Adds automated capacity changes |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- Removing scale-in protection may terminate instances serving traffic
- MinSize reduction may cause immediate capacity loss

## Data Sensitivity

- Scale-in protection settings reveal which instances are considered critical
- Instance lifecycle states reveal deployment and operational patterns
- Scaling activities reveal capacity management decisions

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER remove scale-in protection from all instances simultaneously in production
- NEVER reduce MinSize below the minimum needed to serve current traffic

## Phase 3 — Rollback

- If DisableScaleIn was changed: restore original policy setting
- If scale-in protection was removed: re-enable on critical instances using `aws autoscaling set-instance-protection`
- If MinSize was reduced: restore original MinSize value
- If Terminate process was resumed: re-suspend if causing issues
- Document all changes for operational audit trail

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
