---
title: "B1 — Instance Launch Errors"
description: "Diagnose EC2 instance launch failures in Local Zones"
status: active
severity: HIGH
triggers: ["instance launch failed", "cannot launch in Local Zone", "InsufficientInstanceCapacity", "Unsupported instance type"]
owner: devops-agent
objective: "Resolve EC2 instance launch failures in Local Zones"
context: "Instance launches in Local Zones can fail due to limited instance type availability, capacity constraints, or missing subnet configuration."
---

## Phase 1 — Triage

MUST:
- Verify instance type is available in the zone: `aws ec2 describe-instance-type-offerings --location-type availability-zone --filters "Name=location,Values=<local-zone-name>" --query 'InstanceTypeOfferings[*].InstanceType' | grep <instance-type>`
- Check subnet exists in the Local Zone: `aws ec2 describe-subnets --filters "Name=availability-zone,Values=<local-zone-name>" --query 'Subnets[*].{SubnetId:SubnetId,AvailableIpAddressCount:AvailableIpAddressCount}'`
- Review launch error details from console or CLI output

SHOULD:
- Check AMI availability in the region (AMIs are regional, not zone-specific)
- Verify security group is in the same VPC as the Local Zone subnet
- Check for capacity reservations in the Local Zone

MAY:
- Review EC2 instance limits for the account
- Check CloudTrail for RunInstances errors

## Phase 2 — Remediate

MUST:
- Use a supported instance type for the Local Zone
- Launch in a subnet within the Local Zone
- Ensure AMI is available in the parent region

SHOULD:
- Use multiple instance types in launch templates for better availability
- Consider Capacity Reservations for critical workloads

MAY:
- Request capacity increase through AWS Support
- Configure Auto Scaling with mixed instance types

## Safety Ratings

```
safety_ratings:
  - "describe-instance-type-offerings: GREEN — read-only type check"
  - "describe-subnets: GREEN — read-only subnet check"
  - "run-instances: YELLOW — launches compute resources with cost implications"
  - "create-capacity-reservation: YELLOW — reserves capacity with cost commitment"
```

## Escalation Conditions

- InsufficientInstanceCapacity persists for all available instance types
- Critical workload cannot launch in Local Zone requiring capacity request
- Instance type needed is not available in any Local Zone in the metro area
- Launch failures during scaling event affecting production workload

## Data Sensitivity

- Instance type offerings: LOW — public availability information
- Subnet details: MEDIUM — reveals network architecture
- Launch parameters: MEDIUM — may contain user data scripts
- CloudTrail RunInstances events: MEDIUM — reveals instance configuration

## Prohibited Actions

- NEVER launch instances in Local Zones without verifying instance type availability first
- NEVER use Local Zone for workloads requiring instance types not available in the zone
- NEVER ignore InsufficientInstanceCapacity errors — they indicate real capacity limits
- NEVER launch large fleets in Local Zones without capacity planning

## Phase 3 — Rollback

1. If instances were launched with wrong configuration, terminate them: `aws ec2 terminate-instances --instance-ids <id>`
2. If Capacity Reservation was created unnecessarily, cancel it: `aws ec2 cancel-capacity-reservation --capacity-reservation-id <id>`
3. If launch template was modified, revert to previous version
4. Verify no orphaned resources (EBS volumes, ENIs) remain after termination

## Common Issues

- symptoms: "RunInstances returns Unsupported for the requested instance type"
  diagnosis: "Instance type is not available in the Local Zone."
  resolution: "Check available types with describe-instance-type-offerings. Choose a supported type."

- symptoms: "RunInstances returns InsufficientInstanceCapacity"
  diagnosis: "Local Zone has insufficient capacity for the requested instance type."
  resolution: "Try a different instance type. Retry later. Contact AWS Support for capacity requests."

## Output Format

```yaml
root_cause: "instance_launch — <specific_cause>"
evidence:
  - type: launch_error
    content: "<error details and instance type availability>"
severity: HIGH
mitigation:
  immediate: "Use available instance type"
  long_term: "Implement capacity planning and mixed instance types"
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
