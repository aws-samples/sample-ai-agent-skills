---
title: "E2 — Capacity Planning"
description: "Diagnose capacity planning and reservation issues in Local Zones"
status: active
severity: MEDIUM
triggers: ["capacity planning", "InsufficientCapacity", "capacity reservation", "scaling limits", "cannot scale"]
owner: devops-agent
objective: "Resolve capacity planning issues for Local Zone deployments"
context: "Local Zones have more constrained capacity than full regions. Capacity planning is critical for production workloads. Capacity Reservations may be available for some instance types."
---

## Phase 1 — Triage

MUST:
- Check available instance types and their families: `aws ec2 describe-instance-type-offerings --location-type availability-zone --filters "Name=location,Values=<local-zone-name>"`
- Check existing capacity reservations: `aws ec2 describe-capacity-reservations --filters "Name=availability-zone,Values=<local-zone-name>"`
- Review current instance count in the zone: `aws ec2 describe-instances --filters "Name=availability-zone,Values=<local-zone-name>" "Name=instance-state-name,Values=running" --query 'Reservations[*].Instances[*].InstanceType' --output text | sort | uniq -c`

SHOULD:
- Check EC2 instance limits for the account
- Review Auto Scaling group configurations targeting the Local Zone
- Analyze historical capacity usage patterns

MAY:
- Check if On-Demand Capacity Reservations are supported for needed types
- Review AWS Compute Optimizer recommendations

## Phase 2 — Remediate

MUST:
- Use multiple instance types in launch configurations for better availability
- Create Capacity Reservations for critical workloads (if supported)
- Right-size instances to maximize available capacity

SHOULD:
- Configure Auto Scaling with mixed instance types policy
- Set up CloudWatch alarms for capacity-related failures

MAY:
- Request capacity increase through AWS Support for planned scaling events
- Implement overflow to parent region when Local Zone capacity is exhausted

## Safety Ratings

```
safety_ratings:
  - "describe-instance-type-offerings: GREEN — read-only availability check"
  - "describe-capacity-reservations: GREEN — read-only reservation check"
  - "create-capacity-reservation: YELLOW — reserves capacity with cost commitment"
  - "cancel-capacity-reservation: YELLOW — releases reserved capacity"
  - "modify-capacity-reservation: YELLOW — changes reservation parameters"
```

## Escalation Conditions

- Capacity Reservations not available for required instance types
- Planned scaling event exceeds estimated Local Zone capacity
- InsufficientInstanceCapacity errors during production scaling
- Need to coordinate capacity with AWS for large deployments

## Data Sensitivity

- Capacity reservation details: LOW — reservation metadata
- Instance count and types: MEDIUM — reveals deployment scale
- Auto Scaling configuration: MEDIUM — reveals scaling strategy
- Account limits: LOW — service quota information

## Prohibited Actions

- NEVER create large Capacity Reservations without cost approval
- NEVER plan for unlimited scaling in Local Zones
- NEVER ignore InsufficientInstanceCapacity errors as transient without investigation
- NEVER deploy production workloads without a capacity overflow strategy

## Phase 3 — Rollback

1. If Capacity Reservation was created unnecessarily, cancel it: `aws ec2 cancel-capacity-reservation --capacity-reservation-id <id>`
2. If Auto Scaling was configured with wrong instance types, update the group
3. If overflow to parent region was configured incorrectly, revert routing
4. Document capacity decisions and update planning documents

## Common Issues

- symptoms: "Auto Scaling cannot launch instances during peak hours"
  diagnosis: "Local Zone capacity exhausted for the requested instance type."
  resolution: "Use mixed instance types in ASG. Create Capacity Reservations. Implement parent region overflow."

- symptoms: "Capacity Reservation shows 0 available instances"
  diagnosis: "All reserved capacity is in use or reservation expired."
  resolution: "Check reservation utilization. Increase reservation count or add new reservations."

## Output Format

```yaml
root_cause: "capacity_planning — <specific_cause>"
evidence:
  - type: capacity_data
    content: "<capacity reservation and instance count details>"
severity: MEDIUM
mitigation:
  immediate: "Use mixed instance types"
  long_term: "Implement capacity planning with reservations and overflow"
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
