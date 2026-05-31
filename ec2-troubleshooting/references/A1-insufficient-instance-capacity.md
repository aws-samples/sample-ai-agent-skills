---
title: "A1 — InsufficientInstanceCapacity"
description: "Diagnose and work around EC2 launch failures due to insufficient capacity in the target AZ"
status: active
severity: HIGH
triggers:
  - "InsufficientInstanceCapacity"
  - "We currently do not have sufficient.*capacity"
  - "insufficient capacity"
owner: devops-agent
objective: "Identify the capacity constraint and launch the instance successfully"
context: "EC2 capacity is per-AZ and per-instance-type. When an AZ runs low on a specific instance type, launches fail with InsufficientInstanceCapacity. This is an AWS-side constraint, not a customer configuration error."
---

## Phase 1 — Triage

MUST:
- Confirm the error message is InsufficientInstanceCapacity (not a limit or quota error)
- Identify the instance type and AZ from the failed launch request
- Check if this is a single instance or part of an Auto Scaling group / fleet request
- Check CloudTrail for the RunInstances event to get exact error details

SHOULD:
- Check if the same instance type is available in other AZs in the region
- Check if On-Demand Capacity Reservations exist for this instance type/AZ

MAY:
- Check EC2 instance type offerings per AZ: `aws ec2 describe-instance-type-offerings --location-type availability-zone --filters Name=instance-type,Values=<type>`

## Phase 2 — Remediate

MUST:
- Try launching in a different AZ (most common fix)
- If using Auto Scaling: ensure multiple AZs are configured in the ASG
- If using Spot: add multiple instance types to the Spot Fleet/ASG mixed instances policy

SHOULD:
- Consider using a different instance type in the same family (e.g., m5.xlarge → m5.2xlarge or m5a.xlarge)
- For critical workloads: use On-Demand Capacity Reservations to guarantee capacity
- For Spot: use capacity-optimized allocation strategy

MAY:
- Try a different instance generation (m5 → m6i, c5 → c6i)
- Consider a different region if all AZs in the current region are exhausted

## Guardrails

escalation_conditions:
  - "All AZs in the region return InsufficientInstanceCapacity for the required type"
  - "Capacity Reservation request also fails"
  - "Critical production workload cannot launch anywhere"

safety_ratings:
  - "Launching in different AZ: GREEN"
  - "Changing instance type: YELLOW — verify application compatibility"
  - "Changing region: RED — requires architecture review"

## Common Issues

- symptoms: "RunInstances returns InsufficientInstanceCapacity for a specific AZ"
  diagnosis: "AZ-specific capacity shortage for the requested instance type."
  resolution: "Launch in a different AZ. If using ASG, add more AZs to the subnet list."

- symptoms: "Spot request returns InsufficientInstanceCapacity"
  diagnosis: "Spot capacity pool exhausted for this instance type/AZ."
  resolution: "Add more instance types to the Spot request. Use capacity-optimized allocation strategy."

- symptoms: "Capacity shortage persists across all AZs for a specific instance type"
  diagnosis: "Region-wide capacity constraint for this instance type. Often affects older or very large instance types."
  resolution: "Use a different instance type or generation. Contact AWS support for capacity planning."

## Output Format

```yaml
root_cause: "InsufficientInstanceCapacity — <instance_type> in <az>"
evidence:
  - type: cloudtrail_event
    content: "<RunInstances error from CloudTrail>"
severity: HIGH
mitigation:
  immediate: "Launch in different AZ or use alternative instance type"
  long_term: "Use Capacity Reservations, multi-AZ ASG, or diversified Spot fleet"
```

## Data Sensitivity

data_sensitivity:
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
