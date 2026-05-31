---
title: "A3 — Scaling Too Aggressively"
description: "Diagnose why an Auto Scaling group is scaling up and down too rapidly"
status: active
severity: MEDIUM
triggers:
  - "scaling too fast"
  - "oscillating"
  - "thrashing"
  - "scaling up and down"
  - "too many scaling events"
owner: devops-agent
objective: "Identify why the ASG is scaling too aggressively and stabilize it"
context: "Aggressive scaling or oscillation occurs when: cooldown is too short, metric is too volatile, alarm evaluation period is too short, multiple conflicting policies exist, or external tools are modifying desired capacity."
---

## Phase 1 — Triage

MUST:
- Check recent scaling activities for rapid changes: `aws autoscaling describe-scaling-activities --auto-scaling-group-name <asg-name> --max-items 30`
- Check all scaling policies: `aws autoscaling describe-policies --auto-scaling-group-name <asg-name>`
- Check cooldown settings: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> --query 'AutoScalingGroups[*].DefaultCooldown'`
- Check CloudWatch alarm configuration: `aws cloudwatch describe-alarms --alarm-names <alarm-names> --query 'MetricAlarms[*].{Name:AlarmName,Period:Period,EvaluationPeriods:EvaluationPeriods,Threshold:Threshold}'`

SHOULD:
- Check for multiple policies that may conflict
- Review metric volatility in CloudWatch
- Check CloudTrail for external desired capacity changes: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=SetDesiredCapacity --max-results 20`

MAY:
- Analyze scaling patterns over 24-48 hours
- Check for scheduled actions conflicting with dynamic scaling

## Phase 2 — Remediate

MUST:
- Increase cooldown period if too short: `aws autoscaling update-auto-scaling-group --auto-scaling-group-name <asg-name> --default-cooldown 600`
- Use target tracking instead of simple scaling (it manages cooldown automatically)
- Increase alarm evaluation periods to smooth out spikes

SHOULD:
- Use step scaling with appropriate step adjustments instead of simple scaling
- Add scale-in cooldown to target tracking policies
- Remove conflicting policies

MAY:
- Implement predictive scaling for recurring patterns
- Use composite alarms to reduce false triggers

## Output Format

```yaml
root_cause: "scaling_oscillation — <specific_cause>"
evidence:
  - type: scaling_activities
    content: "<recent rapid scaling events>"
  - type: cooldown_config
    content: "<cooldown settings>"
severity: MEDIUM
mitigation:
  immediate: "Increase cooldown or switch to target tracking"
  long_term: "Implement stable scaling strategy with appropriate thresholds"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Increase cooldown period | YELLOW | Changes scaling responsiveness — may delay needed scaling |
| Switch to target tracking | YELLOW | Changes scaling strategy — test before production |
| Increase alarm evaluation periods | YELLOW | Changes alarm sensitivity — may miss real events |
| Remove conflicting policies | RED | Removes scaling behavior — may prevent needed scaling |
| Add scale-in cooldown | YELLOW | Changes scale-in responsiveness |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- Cooldown changes may delay response to legitimate load spikes
- Removing scaling policies may leave ASG unable to respond to demand

## Data Sensitivity

- Scaling activity patterns reveal traffic and load patterns
- CloudWatch alarm configurations reveal operational thresholds
- Cooldown settings reveal application startup characteristics

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER remove all scaling policies simultaneously
- NEVER set cooldown so high that the ASG cannot respond to legitimate demand

## Phase 3 — Rollback

- If cooldown was increased too much: restore original cooldown value
- If scaling policy was changed to target tracking: restore previous policy type
- If alarm evaluation periods were changed: restore original alarm configuration
- If conflicting policies were removed: recreate them if needed
- Document all scaling policy changes for operational audit trail

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
