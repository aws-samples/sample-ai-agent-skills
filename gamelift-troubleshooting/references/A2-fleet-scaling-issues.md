---
title: "A2 — Fleet Scaling Issues"
description: "Diagnose GameLift fleet scaling issues"
status: active
severity: HIGH
triggers: ["scaling not working", "fleet not scaling", "too many instances", "not enough capacity", "auto-scaling"]
owner: devops-agent
objective: "Resolve fleet auto-scaling and manual scaling issues"
context: "GameLift supports target-based scaling, rule-based scaling, and manual scaling. Scaling reacts to CloudWatch metrics. Instance launch takes minutes."
---

## Phase 1 — Triage

MUST:
- Check current capacity: `aws gamelift describe-fleet-capacity --fleet-id <fleet-id>`
- List scaling policies: `aws gamelift describe-scaling-policies --fleet-id <fleet-id>`
- Check CloudWatch metrics: `aws cloudwatch get-metric-statistics --namespace AWS/GameLift --metric-name AvailableGameSessions --dimensions Name=FleetId,Value=<fleet-id> --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Verify scaling limits (min/max): `aws gamelift describe-fleet-attributes --fleet-ids <fleet-id> --query 'FleetAttributes[0].{Min:InstanceRoleCredentialsProvider}'`
- Check fleet utilization: `aws gamelift describe-fleet-utilization --fleet-ids <fleet-id>`

MAY:
- Review scaling events in fleet events: `aws gamelift describe-fleet-events --fleet-id <fleet-id> --start-time <start> --end-time <end>`
- Check EC2 instance limits in the region

## Phase 2 — Remediate

MUST:
- For target-based scaling: `aws gamelift put-scaling-policy --fleet-id <fleet-id> --name target-policy --scaling-adjustment-type PercentChangeInCapacity --policy-type TargetBased --target-configuration TargetValue=<value>`
- Set appropriate min/max instance limits: `aws gamelift update-fleet-capacity --fleet-id <fleet-id> --min-size <min> --max-size <max>`
- For immediate capacity: manual scaling: `aws gamelift update-fleet-capacity --fleet-id <fleet-id> --desired-instances <count>`

SHOULD:
- Use PercentAvailableGameSessions metric for target-based scaling
- Set buffer capacity (e.g., 20% available sessions) for responsive scaling

MAY:
- Combine target-based and rule-based policies for complex scenarios
- Use scheduled scaling for predictable demand patterns

## Safety Ratings

```
safety_ratings:
  - "describe-fleet-capacity / describe-scaling-policies: GREEN — read-only inspection"
  - "put-scaling-policy: YELLOW — modifies scaling behavior"
  - "update-fleet-capacity: YELLOW — changes instance count, affects costs"
  - "delete-scaling-policy: YELLOW — removes auto-scaling, fleet won't scale automatically"
```

## Escalation Conditions

- Fleet at maximum capacity and still insufficient for demand
- EC2 instance limits preventing scaling in the region
- Scaling policies conflicting with each other
- Spot capacity unavailable for Spot fleets

## Data Sensitivity

- `aws gamelift describe-fleet-capacity`: LOW — instance counts
- `aws gamelift describe-scaling-policies`: LOW — scaling configuration
- `aws cloudwatch get-metric-statistics`: LOW — utilization metrics
- Fleet utilization data: LOW — session and player counts

## Prohibited Actions

- NEVER set max instances to unlimited without cost controls
- NEVER delete all scaling policies without setting manual capacity
- NEVER scale to 0 instances during active game sessions
- NEVER use aggressive scale-down policies that terminate active sessions

## Phase 3 — Rollback

1. If scaling policy was misconfigured: `aws gamelift delete-scaling-policy --fleet-id <id> --name <policy-name>` and recreate
2. If manual scaling set wrong count: `aws gamelift update-fleet-capacity --fleet-id <id> --desired-instances <correct-count>`
3. If min/max limits were wrong: restore original limits
4. If scaling caused cost overrun: scale down and review policies
5. Document scaling changes and update monitoring alerts

## Common Issues

- symptoms: "Fleet has auto-scaling policy but instance count never changes"
  diagnosis: "Scaling policy target metric is not being published, or min/max limits prevent scaling."
  resolution: "Verify CloudWatch metrics are being published. Check min-size and max-size limits. Ensure scaling policy threshold is appropriate."

## Output Format

```yaml
root_cause: "fleet_scaling — <specific_cause>"
evidence:
  - type: scaling_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix scaling configuration"
  long_term: "Implement target-based scaling with appropriate buffer and monitoring"
```

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
