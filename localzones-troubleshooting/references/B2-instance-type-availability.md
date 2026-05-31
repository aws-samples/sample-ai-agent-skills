---
title: "B2 — Instance Type Availability"
description: "Diagnose instance type availability and selection issues in Local Zones"
status: active
severity: MEDIUM
triggers: ["instance type not available", "which instance types", "no GPU instances", "instance family not supported"]
owner: devops-agent
objective: "Identify available instance types and resolve selection issues"
context: "Each Local Zone supports a different subset of instance types. Availability varies by zone and changes over time as AWS expands capacity."
---

## Phase 1 — Triage

MUST:
- List all available instance types: `aws ec2 describe-instance-type-offerings --location-type availability-zone --filters "Name=location,Values=<local-zone-name>" --query 'InstanceTypeOfferings[*].InstanceType' --output table`
- Compare with required instance type
- Check if alternative instance types in the same family are available

SHOULD:
- Check instance type details: `aws ec2 describe-instance-types --instance-types <type> --query 'InstanceTypes[0].{vCPUs:VCpuInfo.DefaultVCpus,Memory:MemoryInfo.SizeInMiB,GPU:GpuInfo}'`
- Verify pricing for available types in the Local Zone

MAY:
- Check multiple Local Zones in the same metro area for broader type availability
- Review AWS documentation for planned instance type additions

## Phase 2 — Remediate

MUST:
- Select an available instance type that meets workload requirements
- Verify the selected type has sufficient vCPUs, memory, and network performance

SHOULD:
- Configure launch templates with multiple instance types for resilience
- Document available types for the team

MAY:
- Request specific instance type availability through AWS Support
- Consider parent region for workloads requiring unavailable types

## Safety Ratings

```
safety_ratings:
  - "describe-instance-type-offerings: GREEN — read-only availability check"
  - "describe-instance-types: GREEN — read-only type specifications"
  - "run-instances with new type: YELLOW — launches resources with cost implications"
  - "modify-instance-attribute: YELLOW — changes instance configuration"
```

## Escalation Conditions

- Required instance type (e.g., GPU) is not available in any Local Zone in the metro
- Workload cannot be adapted to available instance types
- Instance type was previously available but is no longer offered
- Capacity Reservation needed for a specific type not available for reservation

## Data Sensitivity

- Instance type offerings: LOW — public availability information
- Instance type specifications: LOW — public hardware specifications
- Workload requirements: MEDIUM — reveals application architecture
- Pricing information: LOW — publicly available pricing data

## Prohibited Actions

- NEVER assume instance type availability without checking the specific Local Zone
- NEVER deploy production workloads on instance types without verifying Local Zone support
- NEVER ignore instance generation differences (m5 vs m6i) when selecting alternatives
- NEVER commit to capacity reservations without confirming type availability in the zone

## Phase 3 — Rollback

1. If workload was migrated to wrong instance type, launch new instance with correct type
2. If launch template was updated with unavailable type, revert to previous version
3. If Auto Scaling group was configured with unsupported types, update instance requirements
4. Verify application performance on the selected instance type before decommissioning old instances

## Common Issues

- symptoms: "Need GPU instances but g4dn not available in the Local Zone"
  diagnosis: "Not all Local Zones support GPU instance types."
  resolution: "Check other Local Zones in the metro area. Consider parent region for GPU workloads."

- symptoms: "Instance type was available last month but launches fail now"
  diagnosis: "Capacity constraints or instance type retirement in the zone."
  resolution: "Check current offerings. Try alternative types. Contact AWS Support if type was recently removed."

## Output Format

```yaml
root_cause: "instance_type_availability — <specific_cause>"
evidence:
  - type: type_offerings
    content: "<available instance types in the zone>"
severity: MEDIUM
mitigation:
  immediate: "Select available instance type"
  long_term: "Implement multi-type launch templates"
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
