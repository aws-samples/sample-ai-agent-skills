---
title: "G1 — Mixed Instances Policy Issues"
description: "Diagnose mixed instances policy configuration and allocation problems"
status: active
severity: MEDIUM
triggers:
  - "mixed instances not working"
  - "instance type override"
  - "on-demand base capacity"
  - "mixed policy error"
  - "instance weighting"
owner: devops-agent
objective: "Identify and resolve mixed instances policy configuration issues"
context: "Mixed instances policies allow using multiple instance types with On-Demand and Spot mix. Issues include: insufficient instance type diversity, weighting misconfigured, base capacity wrong, or allocation strategy suboptimal."
---

## Phase 1 — Triage

MUST:
- Check mixed instances policy: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> --query 'AutoScalingGroups[*].MixedInstancesPolicy'`
- Check instance distribution: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> --query 'AutoScalingGroups[*].Instances[*].{Id:InstanceId,Type:InstanceType,AZ:AvailabilityZone,Lifecycle:LifecycleState}'`
- Verify instance type availability in target AZs: `aws ec2 describe-instance-type-offerings --location-type availability-zone --filters Name=instance-type,Values=<types>`
- Check scaling activities for allocation failures

SHOULD:
- Verify On-Demand base capacity and percentage settings
- Check instance weighting if different sizes are used
- Verify at least 3-4 instance types for Spot diversity

MAY:
- Check Spot pricing history for selected types
- Review capacity-optimized vs lowest-price strategy effectiveness

## Phase 2 — Remediate

MUST:
- Add more instance type diversity (minimum 3-4 types recommended)
- Fix instance weighting for different sizes
- Use capacity-optimized allocation for Spot

SHOULD:
- Update mixed instances policy with proper configuration
- Set appropriate On-Demand base capacity and percentage above base

MAY:
- Use attribute-based instance type selection for automatic diversity
- Monitor Spot interruption rates by instance type

## Output Format

```yaml
root_cause: "mixed_instances_issue — <specific_cause>"
evidence:
  - type: policy_config
    content: "<mixed instances policy>"
  - type: instance_distribution
    content: "<current instance types and lifecycle>"
severity: MEDIUM
mitigation:
  immediate: "Add instance type diversity or fix allocation strategy"
  long_term: "Use attribute-based selection and capacity-optimized strategy"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Add more instance type diversity | YELLOW | Changes instance allocation — affects cost and availability |
| Fix instance weighting | YELLOW | Changes capacity calculation |
| Use capacity-optimized allocation for Spot | YELLOW | Changes Spot allocation strategy |
| Set On-Demand base capacity and percentage | YELLOW | Changes cost and reliability balance |
| Use attribute-based instance type selection | YELLOW | Automatic instance type selection — verify compatibility |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- Mixed instances policy changes affect instance types and cost
- Allocation strategy changes affect Spot interruption risk

## Data Sensitivity

- Mixed instances policy reveals cost optimization strategy
- Instance distribution reveals infrastructure diversity
- Spot vs On-Demand ratio reveals cost management approach

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER set On-Demand base capacity to 0 for critical production workloads
- NEVER use a single instance type for Spot allocations

## Phase 3 — Rollback

- If mixed instances policy was changed: restore original policy configuration
- If instance weighting was modified: restore original weights
- If allocation strategy was changed: revert to previous strategy
- Document all mixed instances policy changes for operational audit trail

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
