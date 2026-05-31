---
title: "G2 — Spot Allocation Issues"
description: "Diagnose Spot instance allocation failures and interruptions"
status: active
severity: MEDIUM
triggers:
  - "Spot capacity unavailable"
  - "Spot interruption"
  - "no Spot instances launching"
  - "Spot price too high"
  - "InsufficientInstanceCapacity"
owner: devops-agent
objective: "Identify and resolve Spot instance allocation issues in Auto Scaling"
context: "Spot allocation issues include: insufficient capacity in selected pools, max price too low, instance type not available as Spot, interruptions causing capacity loss, or allocation strategy not optimal."
---

## Phase 1 — Triage

MUST:
- Check Spot allocation strategy in mixed instances policy: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> --query 'AutoScalingGroups[*].MixedInstancesPolicy.InstancesDistribution'`
- Check for Spot-related launch failures: `aws autoscaling describe-scaling-activities --auto-scaling-group-name <asg-name> --query 'Activities[?contains(StatusMessage,`Spot`) || contains(StatusMessage,`capacity`)].{Time:StartTime,Message:StatusMessage}'`
- Check Spot instance interruption notices: `aws ec2 describe-spot-instance-requests --filters Name=state,Values=active,open --query 'SpotInstanceRequests[*].{Id:SpotInstanceRequestId,Status:Status.Code}'`
- Verify instance types configured for Spot

SHOULD:
- Check Spot price history: `aws ec2 describe-spot-price-history --instance-types <types> --availability-zones <azs> --product-descriptions "Linux/UNIX" --start-time <start>`
- Verify max price setting (if configured)
- Check number of instance type overrides (more = better availability)

MAY:
- Review Spot interruption frequency by instance type
- Check if capacity-optimized strategy is being used

## Phase 2 — Remediate

MUST:
- Add more instance types for Spot diversity (minimum 4-6 recommended)
- Use capacity-optimized or price-capacity-optimized allocation strategy
- Ensure On-Demand base capacity covers minimum requirements

SHOULD:
- Remove max price restriction (let it default to On-Demand price)
- Spread across multiple AZs for better Spot availability
- Use attribute-based instance type selection: specify vCPU and memory ranges

MAY:
- Implement Spot interruption handling with lifecycle hooks
- Use Spot placement score API to find optimal regions/AZs

## Output Format

```yaml
root_cause: "spot_allocation_issue — <specific_cause>"
evidence:
  - type: allocation_strategy
    content: "<Spot allocation configuration>"
  - type: launch_failures
    content: "<Spot-related failures>"
severity: MEDIUM
mitigation:
  immediate: "Add instance type diversity and use capacity-optimized strategy"
  long_term: "Implement Spot best practices with interruption handling"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Add more instance types for Spot diversity | YELLOW | Changes instance allocation — affects availability |
| Use capacity-optimized allocation strategy | YELLOW | Changes Spot allocation strategy |
| Ensure On-Demand base capacity covers minimum | YELLOW | Changes cost and reliability balance |
| Remove max price restriction | YELLOW | May increase Spot costs |
| Use attribute-based instance type selection | YELLOW | Automatic selection — verify compatibility |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- Spot allocation changes affect interruption risk and cost
- On-Demand base capacity changes affect reliability guarantees

## Data Sensitivity

- Spot allocation strategy reveals cost optimization approach
- Instance type selection reveals workload requirements
- Spot price history reveals cost patterns

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER rely entirely on Spot for production workloads without On-Demand fallback
- NEVER use a single instance type for Spot allocations

## Phase 3 — Rollback

- If allocation strategy was changed: restore original strategy in mixed instances policy
- If instance types were modified: restore original instance type list
- If On-Demand base was changed: restore original base capacity setting
- Document all Spot allocation changes for operational audit trail

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
