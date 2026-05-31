---
title: "E2 — Termination Policy Issues"
description: "Diagnose unexpected instance termination behavior during scale-in"
status: active
severity: MEDIUM
triggers:
  - "wrong instance terminated"
  - "termination policy not working"
  - "unexpected termination"
  - "newest instance terminated"
  - "termination order wrong"
owner: devops-agent
objective: "Identify why termination policy is not behaving as expected"
context: "Termination policies determine which instances are terminated during scale-in. Auto Scaling first rebalances across AZs, then applies the termination policy within the selected AZ. This two-step process often causes unexpected behavior."
---

## Phase 1 — Triage

MUST:
- Check termination policies: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> --query 'AutoScalingGroups[*].TerminationPolicies'`
- Check instance distribution across AZs: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> --query 'AutoScalingGroups[*].Instances[*].{Id:InstanceId,AZ:AvailabilityZone,LaunchTemplate:LaunchTemplate}'`
- Check for scale-in protected instances: `aws autoscaling describe-auto-scaling-instances --query 'AutoScalingInstances[?AutoScalingGroupName==`<asg-name>`].{Id:InstanceId,Protected:ProtectedFromScaleIn}'`
- Review scaling activities for termination reasons: `aws autoscaling describe-scaling-activities --auto-scaling-group-name <asg-name>`

SHOULD:
- Understand AZ rebalancing happens BEFORE termination policy
- Check if instances have different launch templates/configurations
- Verify standby instances are excluded from termination

MAY:
- Check for custom termination policy (Lambda function)
- Review instance ages and launch times

## Phase 2 — Remediate

MUST:
- Set appropriate termination policy: `aws autoscaling update-auto-scaling-group --auto-scaling-group-name <asg-name> --termination-policies "OldestInstance"`
- Understand that AZ rebalancing takes priority over termination policy
- Use scale-in protection for critical instances

SHOULD:
- Available policies: Default, OldestInstance, NewestInstance, OldestLaunchConfiguration, OldestLaunchTemplate, ClosestToNextInstanceHour, AllocationStrategy
- Use multiple policies in order of priority

MAY:
- Implement custom termination policy with Lambda
- Use instance protection for stateful workloads

## Output Format

```yaml
root_cause: "termination_policy_issue — <specific_cause>"
evidence:
  - type: termination_policy
    content: "<configured policies>"
  - type: az_distribution
    content: "<instance distribution across AZs>"
severity: MEDIUM
mitigation:
  immediate: "Adjust termination policy or use scale-in protection"
  long_term: "Design for stateless instances that tolerate any termination order"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Set appropriate termination policy | YELLOW | Changes which instances are terminated during scale-in |
| Use scale-in protection for critical instances | GREEN | Protective measure — prevents termination |
| Implement custom termination policy | YELLOW | Custom logic — test thoroughly before production |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- Termination policy changes affect which instances are removed during scale-in
- Custom termination policies require thorough testing

## Data Sensitivity

- Termination policy reveals infrastructure management strategy
- Instance distribution reveals deployment architecture across AZs
- Scale-in protection settings reveal which instances are considered critical

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER change termination policy during active scale-in events
- NEVER remove scale-in protection from stateful instances without data migration

## Phase 3 — Rollback

- If termination policy was changed: restore original policy using `aws autoscaling update-auto-scaling-group`
- If scale-in protection was modified: restore protection on critical instances
- If custom termination policy Lambda was deployed: revert to built-in policy
- Document all termination policy changes for operational audit trail

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
