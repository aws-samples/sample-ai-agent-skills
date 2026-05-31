---
title: "D1 — Auto-Scaling Issues"
description: "Diagnose auto-scaling delays, target utilization, and cooldown period problems"
status: active
severity: HIGH
triggers:
  - "auto-scaling not working"
  - "auto-scaling delay"
  - "capacity not scaling"
  - "scaling too slow"
  - "target utilization"
owner: devops-agent
objective: "Identify and resolve DynamoDB auto-scaling configuration and behavior issues"
context: "DynamoDB auto-scaling uses Application Auto Scaling with target tracking policies. It monitors consumed capacity and adjusts provisioned capacity to maintain a target utilization (default 70%). Scaling out takes 1-2 minutes; scaling in has a default cooldown of 15 minutes. Auto-scaling cannot react to instantaneous spikes — it responds to sustained load changes. Burst capacity covers short spikes."
---

## Phase 1 — Triage

MUST:
- Check auto-scaling configuration: `aws application-autoscaling describe-scalable-targets --service-namespace dynamodb --resource-ids table/<table>`
- Check scaling policies: `aws application-autoscaling describe-scaling-policies --service-namespace dynamodb --resource-id table/<table>`
- Check scaling activity history: `aws application-autoscaling describe-scaling-activities --service-namespace dynamodb --resource-id table/<table>`
- Check consumed vs provisioned capacity to see if scaling is keeping up

SHOULD:
- Check CloudWatch alarms created by auto-scaling: `aws cloudwatch describe-alarms --alarm-name-prefix TargetTracking-table/<table>`
- Verify target utilization percentage in the scaling policy
- Check if GSI auto-scaling is also configured: `aws application-autoscaling describe-scalable-targets --service-namespace dynamodb --resource-ids table/<table>/index/<gsi-name>`
- Review scaling cooldown periods

MAY:
- Check if the IAM role for auto-scaling has correct permissions
- Review if scheduled scaling would better handle predictable patterns
- Check if the min/max capacity range is appropriate

## Phase 2 — Remediate

MUST:
- If auto-scaling is not configured, enable it:
  - Register target: `aws application-autoscaling register-scalable-target --service-namespace dynamodb --resource-id table/<table> --scalable-dimension dynamodb:table:ReadCapacityUnits --min-capacity <min> --max-capacity <max>`
  - Create policy: `aws application-autoscaling put-scaling-policy --service-namespace dynamodb --resource-id table/<table> --scalable-dimension dynamodb:table:ReadCapacityUnits --policy-name <policy-name> --policy-type TargetTrackingScaling --target-tracking-scaling-policy-configuration '{"TargetValue":70.0,"PredefinedMetricSpecification":{"PredefinedMetricType":"DynamoDBReadCapacityUtilization"}}'`
- Set appropriate min and max capacity values
- Ensure auto-scaling is configured for both RCU and WCU

SHOULD:
- Lower target utilization (e.g., 50%) for latency-sensitive workloads
- Increase max capacity to handle expected peak traffic
- Configure auto-scaling on all GSIs as well

MAY:
- Use scheduled scaling for predictable traffic patterns: `aws application-autoscaling put-scheduled-action --service-namespace dynamodb --resource-id table/<table> --scalable-dimension dynamodb:table:ReadCapacityUnits --scheduled-action-name <name> --schedule "cron(0 8 * * ? *)" --scalable-target-action MinCapacity=<min>,MaxCapacity=<max>`
- Combine auto-scaling with on-demand mode evaluation
- Set up CloudWatch alarms for scaling activity monitoring

## Common Issues

- symptoms: "Throttling occurs before auto-scaling kicks in"
  diagnosis: "Auto-scaling takes 1-2 minutes to detect and respond. Sudden spikes exhaust burst capacity before scaling completes."
  resolution: "Lower target utilization to maintain headroom. Use burst capacity for short spikes. For predictable spikes, use scheduled scaling."

- symptoms: "Auto-scaling scales up but never scales down"
  diagnosis: "Scale-in cooldown (default 15 minutes) prevents rapid scale-down. Or min capacity is set too high."
  resolution: "Review min capacity setting. Scale-in cooldown is intentional to prevent flapping. Adjust if needed."

- symptoms: "Auto-scaling configured but capacity never changes"
  diagnosis: "The IAM service-linked role may lack permissions, or the scaling policy target is misconfigured."
  resolution: "Verify the AWSServiceRoleForApplicationAutoScaling_DynamoDBTable role exists. Check scaling policy configuration and CloudWatch alarms."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Register scalable target | GREEN | Enables auto-scaling; no immediate capacity change |
| Create target tracking policy | GREEN | Configures scaling behavior; non-destructive |
| Lower target utilization | GREEN | Increases headroom; may increase cost but improves resilience |
| Increase max capacity | GREEN | Raises scaling ceiling; no immediate change |
| Configure scheduled scaling | YELLOW | Pre-sets capacity at specific times; must match traffic patterns accurately |
| Modify min capacity | YELLOW | Changing minimum may cause immediate scale-in if current capacity exceeds new min |

## Escalation Conditions

- Auto-scaling not responding to sustained load increase in production
- Scaling policy misconfiguration causing repeated throttling
- IAM service-linked role missing or lacking permissions
- Auto-scaling configured but CloudWatch alarms not triggering
- GSI auto-scaling not configured alongside table auto-scaling

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-scalable-targets` | LOW | Auto-scaling configuration only |
| `describe-scaling-policies` | LOW | Scaling policy parameters only |
| `describe-scaling-activities` | LOW | Scaling event history only |
| `describe-alarms` (CloudWatch) | LOW | Alarm configuration only |

## Prohibited Actions

- NEVER suggest deleting a table to fix auto-scaling issues
- NEVER suggest disabling encryption to reduce scaling overhead
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER set max capacity lower than current consumed capacity

## Phase 3 — Rollback

If auto-scaling changes cause issues:
1. Revert scaling policy target utilization: `aws application-autoscaling put-scaling-policy --service-namespace dynamodb --resource-id table/<table> --scalable-dimension dynamodb:table:ReadCapacityUnits --policy-name <policy-name> --policy-type TargetTrackingScaling --target-tracking-scaling-policy-configuration '{"TargetValue":<previous-target>,"PredefinedMetricSpecification":{"PredefinedMetricType":"DynamoDBReadCapacityUtilization"}}'`
2. Revert min/max capacity: `aws application-autoscaling register-scalable-target --service-namespace dynamodb --resource-id table/<table> --scalable-dimension dynamodb:table:ReadCapacityUnits --min-capacity <previous-min> --max-capacity <previous-max>`
3. If scheduled scaling was added, delete the scheduled action: `aws application-autoscaling delete-scheduled-action --service-namespace dynamodb --resource-id table/<table> --scalable-dimension dynamodb:table:ReadCapacityUnits --scheduled-action-name <name>`
4. Verify capacity stabilizes and throttle events subside after rollback

## Output Format

```yaml
root_cause: "auto_scaling — <specific_cause>"
evidence:
  - type: scaling_config
    content: "<scalable targets and policies>"
  - type: scaling_activity
    content: "<recent scaling events>"
  - type: capacity_metrics
    content: "<consumed vs provisioned over time>"
severity: HIGH
auto_scaling_analysis:
  configured: "yes | no"
  target_utilization: "<percentage>"
  min_capacity: "<RCU/WCU>"
  max_capacity: "<RCU/WCU>"
  last_scale_out: "<timestamp>"
  last_scale_in: "<timestamp>"
  gsi_scaling_configured: "yes | no"
mitigation:
  immediate: "Configure or fix auto-scaling policies"
  long_term: "Implement scheduled scaling for predictable patterns"
```

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
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest deleting a table to fix issues"
  - "NEVER run full table scans on production tables"
