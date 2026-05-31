---
title: "A2 — Fleet Scaling Issues"
description: "Diagnose AppStream 2.0 fleet auto-scaling problems"
status: active
severity: HIGH
triggers:
  - "fleet not scaling"
  - "insufficient capacity"
  - "scaling policy not working"
  - "too many instances"
owner: devops-agent
objective: "Identify and resolve fleet auto-scaling issues"
context: "Fleet scaling uses CloudWatch metrics and scaling policies. Common issues include incorrect scaling policies, cooldown periods preventing scaling, minimum/maximum capacity limits, and InsufficientCapacityError when demand exceeds capacity."
---

## Phase 1 — Triage

MUST:
- Check fleet capacity: `aws appstream describe-fleets --names <fleet-name> --query 'Fleets[*].{Desired:ComputeCapacityStatus.Desired,Running:ComputeCapacityStatus.Running,InUse:ComputeCapacityStatus.InUse,Available:ComputeCapacityStatus.Available}'`
- Check scaling policies: `aws application-autoscaling describe-scaling-policies --service-namespace appstream --resource-id fleet/<fleet-name>`
- Check InsufficientCapacityError metric: `aws cloudwatch get-metric-statistics --namespace AWS/AppStream --metric-name InsufficientCapacityError --dimensions Name=Fleet,Value=<fleet-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check CapacityUtilization metric: `aws cloudwatch get-metric-statistics --namespace AWS/AppStream --metric-name CapacityUtilization --dimensions Name=Fleet,Value=<fleet-name> --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Verify scaling target is registered: `aws application-autoscaling describe-scalable-targets --service-namespace appstream --resource-ids fleet/<fleet-name>`
- Check cooldown periods in scaling policies
- Verify minimum and maximum capacity settings

MAY:
- Check CloudTrail for scaling events
- Review scaling activity history

## Phase 2 — Remediate

MUST:
- Register scaling target if not registered:
  `aws application-autoscaling register-scalable-target --service-namespace appstream --resource-id fleet/<fleet-name> --scalable-dimension appstream:fleet:DesiredCapacity --min-capacity <min> --max-capacity <max>`
- Create or fix scaling policies based on CapacityUtilization
- Adjust minimum capacity to handle baseline demand

SHOULD:
- Set appropriate cooldown periods (scale-out: 60s, scale-in: 300s)
- Configure step scaling for gradual capacity changes
- Set minimum capacity > 0 for always-available sessions

MAY:
- Set up CloudWatch alarms for InsufficientCapacityError
- Create scheduled scaling for predictable usage patterns

## Common Issues

- symptoms: "Users getting InsufficientCapacityError"
  diagnosis: "All instances in use and scaling hasn't caught up."
  resolution: "Increase minimum capacity. Reduce scale-out cooldown. Increase maximum capacity."

- symptoms: "Fleet not scaling down when users leave"
  diagnosis: "Scale-in cooldown too long or scale-in policy missing."
  resolution: "Check scale-in policy and cooldown period. Ensure scale-in policy exists."

- symptoms: "Scaling policy exists but not triggering"
  diagnosis: "Scalable target not registered or metric threshold not reached."
  resolution: "Register scalable target. Verify metric threshold matches actual utilization."

## Output Format

```yaml
root_cause: "fleet_scaling_issue — <specific_cause>"
evidence:
  - type: fleet_capacity
    content: "<current capacity status>"
  - type: scaling_policies
    content: "<scaling policy configuration>"
  - type: capacity_metrics
    content: "<CloudWatch capacity metrics>"
severity: HIGH
mitigation:
  immediate: "Fix scaling configuration"
  long_term: "Implement comprehensive scaling with scheduled and dynamic policies"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Register scaling target | YELLOW | Enables auto-scaling |
| Create/fix scaling policies | YELLOW | Changes scaling behavior |
| Adjust minimum capacity | YELLOW | Affects always-available instances |
| Set cooldown periods | GREEN | Scaling tuning |
| Set up CloudWatch alarms | GREEN | Monitoring setup |

## Escalation Conditions

- Fix requires fleet rebuild affecting active users

## Data Sensitivity

- **Fleet capacity**: Running, desired, in-use instance counts — INTERNAL
- **Scaling policies**: Scaling thresholds and cooldowns — INTERNAL
- **Capacity metrics**: CloudWatch utilization data — INTERNAL

## Prohibited Actions

- NEVER suggest deleting fleet with active sessions
- NEVER set minimum capacity to 0 during business hours
- NEVER modify scaling policies during peak usage without approval

## Phase 3 — Rollback

1. Revert scaling policies to previous configuration
2. Restore previous minimum and maximum capacity settings
3. Deregister scalable target if auto-scaling was newly enabled
4. Verify fleet capacity returns to expected levels
5. Monitor InsufficientCapacityError metrics after rollback

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
