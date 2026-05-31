---
title: "A2 — Outpost Capacity Exhaustion"
description: "Diagnose and resolve capacity issues on AWS Outposts"
status: active
severity: HIGH
triggers:
  - "InsufficientInstanceCapacity"
  - "capacity full"
  - "cannot launch instance"
  - "no capacity"
owner: devops-agent
objective: "Identify capacity constraints and optimize resource utilization on the Outpost"
context: "Outpost capacity is physically limited by installed hardware. Unlike Regions, capacity does not elastically scale. When an instance family is exhausted, launches fail until capacity is freed or additional hardware is ordered."
---

## Phase 1 — Triage

MUST:
- Check capacity availability: `aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name InstanceFamilyCapacityAvailability --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- List available instance types: `aws outposts get-outpost-instance-types --outpost-id <outpost-id>`
- List running instances on the Outpost: `aws ec2 describe-instances --filters "Name=outpost-arn,Values=<outpost-arn>" "Name=instance-state-name,Values=running" --query 'Reservations[*].Instances[*].{Id:InstanceId,Type:InstanceType,State:State.Name}'`
- Check Outpost assets: `aws outposts list-assets --outpost-identifier <outpost-id>`

SHOULD:
- Identify underutilized instances that could be downsized or terminated
- Check for stopped instances consuming capacity reservations
- Review instance type distribution across families

MAY:
- Analyze historical capacity trends to predict future needs
- Check if workloads can use alternative instance families with available capacity

## Phase 2 — Remediate

MUST:
- Terminate or stop unused instances to free capacity
- Right-size over-provisioned instances to smaller types
- If capacity is genuinely exhausted, contact AWS to order additional hardware

SHOULD:
- Implement capacity monitoring with CloudWatch alarms
- Use instance scheduling to stop non-production instances outside business hours
- Consider using different instance families if the target family is full

MAY:
- Implement capacity reservation strategies for critical workloads
- Plan capacity expansion with AWS account team

## Common Issues

- symptoms: "InsufficientInstanceCapacity error when launching instances"
  diagnosis: "The requested instance family has no remaining capacity on the Outpost."
  resolution: "Free capacity by terminating unused instances or try a different instance family."

- symptoms: "Capacity metrics show low availability but few instances running"
  diagnosis: "Stopped instances or capacity reservations may be holding capacity."
  resolution: "Check for stopped instances and unused capacity reservations."

## Output Format

```yaml
root_cause: "capacity — <specific_cause>"
evidence:
  - type: capacity_metrics
    content: "<CloudWatch capacity availability>"
  - type: instance_inventory
    content: "<running instances and types>"
severity: HIGH
mitigation:
  immediate: "Free capacity by terminating unused instances"
  long_term: "Implement capacity monitoring and right-sizing"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — capacity management is operational
- Service Disruption Risk: HIGH — capacity exhaustion prevents new workload deployment
- Compliance Impact: LOW — capacity is an operational concern

## Escalation Conditions
- Outpost capacity fully consumed preventing critical workload deployment
- Capacity allocation does not match expected configuration
- Instance launch failures due to insufficient capacity
- Capacity cannot be reclaimed from terminated instances
- Unexpected capacity consumption by unknown workloads

## Data Sensitivity
- Capacity utilization metrics — INTERNAL
- Instance type availability — INTERNAL
- Workload placement details — INTERNAL
- Capacity reservation configurations — INTERNAL
- Never expose capacity details that reveal business scale

## Prohibited Actions
- Do NOT over-provision capacity without cost approval
- Do NOT ignore capacity warnings on production Outposts
- Do NOT terminate running instances to free capacity without approval
- Do NOT bypass capacity reservation policies
- Do NOT deploy non-essential workloads on capacity-constrained Outposts
- Do NOT ignore capacity planning recommendations

## Phase 3 — Rollback
- Restore previous capacity reservation configurations
- Re-launch terminated instances if capacity was freed incorrectly
- Revert workload placement changes
- Restore capacity allocation policies
- Verify capacity availability after changes
- Document capacity management decisions and their rationale

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
