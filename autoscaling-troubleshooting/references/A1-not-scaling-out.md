---
title: "A1 — Not Scaling Out"
description: "Diagnose why an Auto Scaling group is not adding instances under load"
status: active
severity: HIGH
triggers:
  - "not scaling out"
  - "instances not launching"
  - "capacity not increasing"
  - "ASG stuck at minimum"
  - "scaling not triggered"
owner: devops-agent
objective: "Identify why the ASG is not scaling out and resolve the issue"
context: "Auto Scaling groups may fail to scale out due to: max capacity reached, no scaling policy configured, CloudWatch alarm not triggering, cooldown period active, suspended scaling processes, insufficient permissions, or capacity unavailable in the AZ."
---

## Phase 1 — Triage

MUST:
- Check current ASG state: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> --query 'AutoScalingGroups[*].{Min:MinSize,Max:MaxSize,Desired:DesiredCapacity,Instances:Instances[*].InstanceId}'`
- Check if desired capacity equals max: if DesiredCapacity == MaxSize, the ASG cannot scale further
- Check scaling policies exist: `aws autoscaling describe-policies --auto-scaling-group-name <asg-name>`
- Check for suspended processes: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> --query 'AutoScalingGroups[*].SuspendedProcesses'`
- Check recent scaling activities: `aws autoscaling describe-scaling-activities --auto-scaling-group-name <asg-name> --max-items 10`

SHOULD:
- Check CloudWatch alarm state for scaling policies: `aws cloudwatch describe-alarms --alarm-names <alarm-name>`
- Verify the scaling metric is being published: `aws cloudwatch get-metric-statistics --namespace <namespace> --metric-name <metric> --dimensions Name=AutoScalingGroupName,Value=<asg-name> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check if cooldown is active (recent scaling activity within cooldown window)
- Verify IAM service-linked role exists: `aws iam get-role --role-name AWSServiceRoleForAutoScaling`

MAY:
- Check for scheduled actions that may override: `aws autoscaling describe-scheduled-actions --auto-scaling-group-name <asg-name>`
- Review CloudTrail for manual capacity changes

## Phase 2 — Remediate

MUST:
- If max capacity reached: increase MaxSize
- If no scaling policy: create appropriate policy (target tracking recommended)
- If processes suspended: resume Launch and Terminate processes
- If alarm not triggering: verify metric threshold and evaluation periods

SHOULD:
- Create target tracking policy: `aws autoscaling put-scaling-policy --auto-scaling-group-name <asg-name> --policy-name scale-out --policy-type TargetTrackingScaling --target-tracking-configuration '{"PredefinedMetricSpecification":{"PredefinedMetricType":"ASGAverageCPUUtilization"},"TargetValue":70.0}'`
- Resume suspended processes: `aws autoscaling resume-processes --auto-scaling-group-name <asg-name> --scaling-processes Launch AlarmNotification`

MAY:
- Set up SNS notifications for scaling events
- Enable predictive scaling for recurring patterns

## Output Format

```yaml
root_cause: "not_scaling_out — <specific_cause>"
evidence:
  - type: asg_capacity
    content: "<current min/max/desired>"
  - type: scaling_policies
    content: "<policy configuration>"
  - type: alarm_state
    content: "<CloudWatch alarm state>"
severity: HIGH
mitigation:
  immediate: "Fix the specific scaling blocker"
  long_term: "Implement comprehensive scaling policies with monitoring"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Increase MaxSize | YELLOW | Changes ASG capacity limits — may increase costs |
| Create scaling policy (target tracking) | YELLOW | Adds automated scaling behavior |
| Resume suspended processes | YELLOW | Re-enables scaling — may trigger immediate actions |
| Verify metric threshold and evaluation periods | GREEN | Read-only diagnostic |
| Set up SNS notifications | GREEN | Monitoring — non-destructive |
| Enable predictive scaling | YELLOW | Adds automated pre-provisioning |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- MaxSize increase may significantly impact costs
- Resuming suspended processes may trigger immediate scaling actions

## Data Sensitivity

- ASG configuration reveals infrastructure topology and capacity
- Scaling policies reveal load patterns and thresholds
- CloudWatch metrics may reveal traffic patterns and business activity levels
- Instance IDs and configurations may reveal deployment architecture

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER resume all suspended processes simultaneously without understanding why they were suspended
- NEVER modify scaling policies during active incidents without rollback plan

## Phase 3 — Rollback

- If MaxSize was increased: restore original MaxSize using `aws autoscaling update-auto-scaling-group`
- If scaling policy was created: delete the policy using `aws autoscaling delete-policy`
- If processes were resumed: re-suspend if needed using `aws autoscaling suspend-processes`
- If predictive scaling was enabled: delete the predictive scaling policy
- Document all ASG changes for operational audit trail

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
