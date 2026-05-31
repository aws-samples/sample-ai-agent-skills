---
title: "E2 — Auto-Scaling"
description: "Diagnose Amazon Keyspaces auto-scaling configuration and behavior issues"
status: active
severity: MEDIUM
triggers:
  - "auto-scaling not working"
  - "scaling policy"
  - "scalable target"
  - "capacity not adjusting"
  - "auto scaling configuration"
owner: devops-agent
objective: "Resolve auto-scaling configuration issues for provisioned Keyspaces tables"
context: "Keyspaces provisioned tables use AWS Application Auto Scaling. Scalable targets and scaling policies must be configured separately for read and write capacity. Auto-scaling reacts to CloudWatch alarms with cooldown periods. It does not handle sudden spikes — there is a delay between alarm trigger and capacity adjustment."
---

## Phase 1 — Triage

MUST:
- Check scalable targets: `aws application-autoscaling describe-scalable-targets --service-namespace cassandra --resource-ids keyspace/<keyspace>/table/<table>`
- Check scaling policies: `aws application-autoscaling describe-scaling-policies --service-namespace cassandra --resource-id keyspace/<keyspace>/table/<table>`
- Check scaling activities: `aws application-autoscaling describe-scaling-activities --service-namespace cassandra --resource-id keyspace/<keyspace>/table/<table> --max-results 10`
- Verify table is in provisioned mode: `aws keyspaces get-table --keyspace-name <keyspace> --table-name <table> --query 'capacitySpecification'`

SHOULD:
- Check CloudWatch alarms created by auto-scaling: `aws cloudwatch describe-alarms --alarm-name-prefix TargetTracking-keyspace/<keyspace>/table/<table>`
- Verify IAM role for auto-scaling has correct permissions
- Check cooldown periods in scaling policies

MAY:
- Review scaling history for patterns
- Test scaling by generating load

## Phase 2 — Remediate

MUST:
- Register scalable targets for both read and write:
  - `aws application-autoscaling register-scalable-target --service-namespace cassandra --resource-id keyspace/<keyspace>/table/<table> --scalable-dimension cassandra:table:ReadCapacityUnits --min-capacity 5 --max-capacity 1000`
  - `aws application-autoscaling register-scalable-target --service-namespace cassandra --resource-id keyspace/<keyspace>/table/<table> --scalable-dimension cassandra:table:WriteCapacityUnits --min-capacity 5 --max-capacity 1000`
- Create target tracking scaling policies with 70% target utilization

SHOULD:
- Set appropriate min/max capacity bounds
- Configure scale-in cooldown (300s default) and scale-out cooldown (0s default)
- Set up CloudWatch alarms for throttling as a safety net

MAY:
- Use step scaling policies for more granular control
- Implement scheduled scaling for predictable traffic patterns

## Common Issues

- symptoms: "Auto-scaling not registered for the table"
  diagnosis: "Scalable targets were never configured for this table."
  resolution: "Register scalable targets and create scaling policies for read and write capacity."

- symptoms: "Scaling too slow — throttling before capacity increases"
  diagnosis: "Auto-scaling has inherent delay (alarm evaluation + cooldown + API call)."
  resolution: "Increase minimum capacity. Reduce scale-out cooldown. Consider on-demand for spiky workloads."

- symptoms: "Scaling down too aggressively"
  diagnosis: "Scale-in cooldown too short or target utilization too high."
  resolution: "Increase scale-in cooldown period. Lower target utilization percentage."

## Output Format

```yaml
root_cause: "auto_scaling — <specific_cause>"
evidence:
  - type: scalable_targets
    content: "<registered targets and min/max capacity>"
  - type: scaling_policies
    content: "<policy configuration and target utilization>"
  - type: scaling_activities
    content: "<recent scaling events>"
severity: MEDIUM
mitigation:
  immediate: "Configure or fix auto-scaling targets and policies"
  long_term: "Monitor scaling behavior and adjust parameters"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never drop keyspaces or tables containing production data
- Never modify capacity mode on tables during peak traffic without load testing
- Never alter TTL settings without understanding data retention requirements
- Never change encryption settings on tables with active workloads
- Never disable point-in-time recovery on production tables

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore tables from point-in-time recovery if data is accidentally modified
- Revert capacity mode changes if throttling increases after modification
- Re-enable previous auto-scaling configurations if performance degrades

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
