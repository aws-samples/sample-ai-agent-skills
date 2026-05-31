---
title: "A4 — Cooldown Issues"
description: "Diagnose cooldown-related scaling delays and misconfigurations"
status: active
severity: MEDIUM
triggers:
  - "cooldown blocking scaling"
  - "scaling delayed"
  - "cooldown too long"
  - "cooldown not working"
  - "scaling not responding"
owner: devops-agent
objective: "Identify and resolve cooldown-related scaling issues"
context: "Cooldown prevents Auto Scaling from launching or terminating additional instances before previous scaling activities take effect. Default is 300 seconds. Target tracking has independent cooldowns. Step scaling can disable cooldown. Misconfigured cooldowns cause delayed responses or oscillation."
---

## Phase 1 — Triage

MUST:
- Check default cooldown: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> --query 'AutoScalingGroups[*].DefaultCooldown'`
- Check policy-specific cooldowns: `aws autoscaling describe-policies --auto-scaling-group-name <asg-name> --query 'ScalingPolicies[*].{Name:PolicyName,Type:PolicyType,Cooldown:Cooldown}'`
- Check target tracking cooldowns: `aws autoscaling describe-policies --auto-scaling-group-name <asg-name> --query 'ScalingPolicies[?PolicyType==`TargetTrackingScaling`].TargetTrackingConfiguration'`
- Review recent scaling activities with timestamps: `aws autoscaling describe-scaling-activities --auto-scaling-group-name <asg-name> --max-items 10`

SHOULD:
- Calculate time between scaling events to identify cooldown gaps
- Check if step scaling policies have cooldown disabled
- Verify scheduled actions are not conflicting with cooldown

MAY:
- Compare scaling response time with application startup time
- Review CloudWatch metric resolution vs cooldown period

## Phase 2 — Remediate

MUST:
- Adjust default cooldown based on application startup time
- For target tracking: adjust ScaleInCooldown and ScaleOutCooldown independently
- For step scaling: consider disabling cooldown if rapid response needed

SHOULD:
- Set scale-out cooldown shorter than scale-in cooldown for responsiveness
- Align cooldown with health check grace period

MAY:
- Use warm pools to reduce dependency on cooldown timing
- Implement predictive scaling to pre-provision capacity

## Output Format

```yaml
root_cause: "cooldown_issue — <specific_cause>"
evidence:
  - type: cooldown_config
    content: "<default and policy cooldowns>"
  - type: scaling_timeline
    content: "<scaling activity timestamps>"
severity: MEDIUM
mitigation:
  immediate: "Adjust cooldown values for the specific policy type"
  long_term: "Align cooldown strategy with application characteristics"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Adjust default cooldown | YELLOW | Changes scaling responsiveness for all policies |
| Adjust ScaleInCooldown/ScaleOutCooldown | YELLOW | Changes per-direction scaling responsiveness |
| Disable cooldown for step scaling | YELLOW | Enables rapid scaling — may cause oscillation |
| Use warm pools | YELLOW | Adds pre-initialized instances — increases cost |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- Cooldown changes affect how quickly the ASG responds to all scaling events
- Disabling cooldown may cause scaling oscillation

## Data Sensitivity

- Cooldown configuration reveals application startup and stabilization characteristics
- Scaling activity timestamps reveal operational patterns

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER set cooldown to 0 without understanding oscillation risk
- NEVER modify cooldown during active scaling events

## Phase 3 — Rollback

- If cooldown was changed: restore original cooldown values using `aws autoscaling update-auto-scaling-group`
- If step scaling cooldown was disabled: re-enable cooldown on the policy
- If warm pool was added: remove with `aws autoscaling delete-warm-pool` (instances will be terminated)
- Document all cooldown changes for operational audit trail

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
