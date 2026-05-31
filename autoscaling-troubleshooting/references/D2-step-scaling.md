---
title: "D2 — Step Scaling Policy Issues"
description: "Diagnose step scaling policy not triggering or misconfigured"
status: active
severity: MEDIUM
triggers:
  - "step scaling not working"
  - "step adjustment wrong"
  - "step policy not triggering"
  - "scaling steps misconfigured"
  - "alarm not triggering scaling"
owner: devops-agent
objective: "Identify and fix step scaling policy configuration issues"
context: "Step scaling policies use CloudWatch alarms to trigger scaling with different adjustments based on alarm breach magnitude. Issues include: alarm not in ALARM state, step adjustments misconfigured, metric aggregation type wrong, or cooldown preventing action."
---

## Phase 1 — Triage

MUST:
- Check step scaling policies: `aws autoscaling describe-policies --auto-scaling-group-name <asg-name> --policy-types StepScaling`
- Check associated CloudWatch alarm: `aws cloudwatch describe-alarms --alarm-names <alarm-name>`
- Verify alarm is in correct state: `aws cloudwatch describe-alarms --alarm-names <alarm-name> --query 'MetricAlarms[*].{Name:AlarmName,State:StateValue,Threshold:Threshold}'`
- Check step adjustments configuration

SHOULD:
- Verify metric aggregation type matches alarm statistic
- Check if cooldown is preventing scaling actions
- Verify alarm evaluation periods and datapoints

MAY:
- Test alarm by setting a lower threshold temporarily
- Review alarm history: `aws cloudwatch describe-alarm-history --alarm-name <alarm-name>`

## Phase 2 — Remediate

MUST:
- Fix alarm configuration if not triggering
- Correct step adjustments for appropriate scaling response
- Ensure metric aggregation type is consistent

SHOULD:
- Create step scaling policy: `aws autoscaling put-scaling-policy --auto-scaling-group-name <asg-name> --policy-name step-scale-out --policy-type StepScaling --adjustment-type ChangeInCapacity --step-adjustments '[{"MetricIntervalLowerBound":0,"MetricIntervalUpperBound":20,"ScalingAdjustment":1},{"MetricIntervalLowerBound":20,"ScalingAdjustment":2}]'`

MAY:
- Consider migrating to target tracking for simpler management
- Use composite alarms for more precise triggering

## Output Format

```yaml
root_cause: "step_scaling_issue — <specific_cause>"
evidence:
  - type: policy_config
    content: "<step adjustments>"
  - type: alarm_state
    content: "<alarm state and threshold>"
severity: MEDIUM
mitigation:
  immediate: "Fix alarm or step adjustment configuration"
  long_term: "Consider target tracking for simpler scaling management"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix alarm configuration | YELLOW | Changes when scaling triggers |
| Correct step adjustments | YELLOW | Changes how many instances are added/removed |
| Ensure metric aggregation consistency | GREEN | Configuration alignment — non-destructive |
| Create step scaling policy | YELLOW | Adds new scaling behavior |
| Migrate to target tracking | YELLOW | Changes scaling strategy entirely |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- Step adjustment changes directly affect instance count during scaling events
- Alarm threshold changes affect scaling trigger sensitivity

## Data Sensitivity

- Step scaling configuration reveals operational thresholds and capacity planning
- Alarm history reveals scaling event patterns
- Metric data reveals application load characteristics

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER set step adjustments that could scale to 0 instances
- NEVER modify alarms that are shared with other monitoring systems

## Phase 3 — Rollback

- If alarm was modified: restore original alarm configuration using `aws cloudwatch put-metric-alarm`
- If step adjustments were changed: restore original policy using `aws autoscaling put-scaling-policy`
- If policy was migrated to target tracking: recreate original step scaling policy
- Document all policy and alarm changes for operational audit trail

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
