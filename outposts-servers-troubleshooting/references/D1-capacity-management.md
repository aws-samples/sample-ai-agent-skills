---
title: "D1 — Capacity Management"
description: "Diagnose capacity constraints and resource management on Outposts Servers"
status: active
severity: MEDIUM
triggers: ["InsufficientCapacity", "no capacity on outpost", "cannot launch more instances", "storage full on outpost"]
owner: devops-agent
objective: "Manage and optimize capacity utilization on Outposts Servers"
context: "Outposts Servers have fixed hardware capacity. Compute and storage cannot be added dynamically. Capacity planning requires understanding instance type resource consumption."
---

## Phase 1 — Triage

MUST:
- List running instances on Outpost: `aws ec2 describe-instances --filters "Name=placement-group-name,Values=*" --query 'Reservations[*].Instances[?Placement.OutpostArn!=null].{Id:InstanceId,Type:InstanceType,State:State.Name}'`
- Check supported instance types and counts: `aws outposts get-outpost-instance-types --outpost-id <outpost-id>`
- List EBS volumes on Outpost: `aws ec2 describe-volumes --filters "Name=status,Values=available,in-use" --query 'Volumes[?OutpostArn!=null].{Id:VolumeId,Size:Size,State:State}'`

SHOULD:
- Calculate current vCPU and memory utilization against server limits
- Check for stopped instances consuming capacity reservations
- Review EBS volume utilization and identify unused volumes

MAY:
- Analyze CloudWatch metrics for instance utilization patterns
- Identify right-sizing opportunities for over-provisioned instances

## Phase 2 — Remediate

MUST:
- Terminate or stop unused instances to free capacity
- Delete unattached EBS volumes to reclaim storage
- Right-size instances to match actual workload requirements

SHOULD:
- Implement capacity monitoring alerts via CloudWatch
- Document capacity allocation per workload

MAY:
- Plan additional Outposts Server orders for capacity expansion
- Implement workload scheduling to optimize capacity usage

## Common Issues

- symptoms: "InsufficientInstanceCapacity when launching new instances"
  diagnosis: "Server physical resources exhausted by running instances."
  resolution: "Review running instances: terminate unused, right-size over-provisioned. Delete unused EBS volumes. Consider ordering additional servers."

## Safety Ratings

```
safety_ratings:
  - "describe-instances: GREEN — read-only instance enumeration"
  - "get-outpost-instance-types: GREEN — read-only capacity check"
  - "TerminateInstances: RED — permanently destroys instances and ephemeral data"
  - "DeleteVolume: RED — permanently destroys volume data"
```

## Escalation Conditions

- Capacity exhaustion blocking critical workload deployment
- No instances can be terminated or right-sized without business impact
- Storage capacity exhausted with all volumes actively in use
- Capacity planning indicates need for additional hardware procurement

## Data Sensitivity

- `aws ec2 describe-instances`: MEDIUM — reveals workload details and instance metadata
- `aws ec2 describe-volumes`: LOW — returns volume metadata
- Capacity utilization data: LOW — aggregate resource consumption
- Workload scheduling details: MEDIUM — reveals business operations patterns

## Prohibited Actions

- NEVER terminate production instances to free capacity without explicit approval
- NEVER delete EBS volumes without confirming data is backed up or no longer needed
- NEVER assume capacity can be added dynamically — Outposts Servers have fixed resources
- NEVER over-commit capacity by launching instances that exceed physical limits

## Phase 3 — Rollback

1. If instances were terminated in error, re-launch from AMIs or backups
2. If EBS volumes were deleted, restore from snapshots if available
3. If instance types were changed for right-sizing, revert to original types
4. Document capacity state before and after changes
5. Verify remaining workloads are functioning correctly

## Output Format

```yaml
root_cause: "capacity_management — <specific_cause>"
evidence:
  - type: capacity_status
    content: "<instance counts, types, and resource utilization>"
severity: MEDIUM
mitigation:
  immediate: "Free capacity or optimize usage"
  long_term: "Implement capacity monitoring and planning"
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
