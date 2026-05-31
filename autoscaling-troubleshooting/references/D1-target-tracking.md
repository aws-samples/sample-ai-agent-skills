---
title: "D1 — Target Tracking Policy Issues"
description: "Diagnose target tracking scaling policy not maintaining target value"
status: active
severity: HIGH
triggers:
  - "target tracking not working"
  - "metric not at target"
  - "target tracking not scaling"
  - "CPU target not maintained"
  - "request count target"
owner: devops-agent
objective: "Identify why target tracking is not maintaining the target value"
context: "Target tracking automatically adjusts capacity to maintain a metric at a target value. Issues include: wrong metric selected, target value too high/low, insufficient instance types, metric not publishing, DisableScaleIn enabled, or conflicting policies."
---

## Phase 1 — Triage

MUST:
- Check target tracking policy config: `aws autoscaling describe-policies --auto-scaling-group-name <asg-name> --policy-types TargetTrackingScaling`
- Check the actual metric value vs target: `aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=AutoScalingGroupName,Value=<asg-name> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check CloudWatch alarms created by target tracking: `aws cloudwatch describe-alarms --alarm-name-prefix TargetTracking-<asg-name>`
- Verify ASG can scale (not at max/min): `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name>`

SHOULD:
- Check if DisableScaleIn is set
- Verify the metric is being published consistently
- Check for conflicting scaling policies

MAY:
- Review custom metric configuration if using custom metrics
- Check ALB request count metric for ALBRequestCountPerTarget

## Phase 2 — Remediate

MUST:
- If metric not publishing: fix metric source
- If at max capacity: increase MaxSize
- If DisableScaleIn preventing scale-in: evaluate and update if needed
- If target value inappropriate: adjust target value

SHOULD:
- Update target tracking policy: `aws autoscaling put-scaling-policy --auto-scaling-group-name <asg-name> --policy-name cpu-target --policy-type TargetTrackingScaling --target-tracking-configuration '{"PredefinedMetricSpecification":{"PredefinedMetricType":"ASGAverageCPUUtilization"},"TargetValue":60.0}'`

MAY:
- Use multiple target tracking policies (most aggressive wins for scale-out)
- Combine with predictive scaling

## Output Format

```yaml
root_cause: "target_tracking_issue — <specific_cause>"
evidence:
  - type: policy_config
    content: "<target tracking configuration>"
  - type: metric_value
    content: "<actual vs target metric>"
severity: HIGH
mitigation:
  immediate: "Fix metric publishing or adjust target value"
  long_term: "Tune target value based on application performance data"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix metric source | YELLOW | Changes what drives scaling decisions |
| Increase MaxSize | YELLOW | Allows more instances — cost impact |
| Update DisableScaleIn setting | YELLOW | Changes scale-in behavior |
| Adjust target value | YELLOW | Changes scaling sensitivity |
| Use multiple target tracking policies | YELLOW | Adds scaling complexity |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- Target value changes directly affect instance count and cost
- Metric source changes affect scaling accuracy

## Data Sensitivity

- Target tracking configuration reveals performance targets and thresholds
- Metric values reveal application load and performance characteristics
- CloudWatch alarm states reveal operational status

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER set target value so low that the ASG scales to maximum unnecessarily
- NEVER delete target tracking policy without a replacement scaling strategy

## Phase 3 — Rollback

- If target value was changed: restore original target value using `aws autoscaling put-scaling-policy`
- If MaxSize was increased: restore original MaxSize
- If DisableScaleIn was changed: restore original setting
- If metric source was changed: restore original metric configuration
- Document all policy changes for operational audit trail

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
