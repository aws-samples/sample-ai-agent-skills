---
title: "E2 — Device Availability"
description: "Diagnose device availability and selection issues"
status: active
severity: LOW
triggers: ["device not available", "which device to use", "device offline", "no QPU available"]
owner: devops-agent
objective: "Resolve device availability questions and selection guidance"
context: "Braket devices have varying availability. QPUs have windows, simulators are on-demand, and not all devices are in all regions."
---

## Phase 1 — Triage

MUST:
- List all available devices: `aws braket search-devices --filters '[{"name":"deviceType","operator":"EQUAL","values":["QPU"]}]'`
- Check specific device status: `aws braket get-device --device-arn <device-arn>`
- Verify the target region supports the desired device

SHOULD:
- List simulator devices: `aws braket search-devices --filters '[{"name":"deviceType","operator":"EQUAL","values":["SIMULATOR"]}]'`
- Check device provider status pages for maintenance windows
- Review device capabilities for workload compatibility

MAY:
- Compare device specifications across providers
- Check historical availability patterns

## Phase 2 — Remediate

MUST:
- If target device is offline, identify alternative devices with compatible capabilities
- Verify circuit compatibility with alternative devices
- Confirm regional availability of the selected device

SHOULD:
- Use simulators for immediate execution when QPU is unavailable
- Document device selection criteria for the workload
- Plan task submission around known availability windows

MAY:
- Implement multi-device fallback logic in application code
- Subscribe to AWS notifications for device status changes

## Safety Ratings

```
safety_ratings:
  - "search-devices: GREEN — read-only device enumeration"
  - "get-device: GREEN — read-only device details"
  - "create-quantum-task on alternative device: YELLOW — incurs charges, results may differ"
```

## Escalation Conditions

- No QPU devices available in any region for an extended period
- Device capabilities change without notice affecting existing workloads
- Device listed as ONLINE but not accepting tasks
- Regional availability does not match AWS documentation

## Data Sensitivity

- `aws braket search-devices`: LOW — returns device metadata
- `aws braket get-device`: LOW — returns device capabilities
- Device availability patterns: LOW — public information

## Prohibited Actions

- NEVER guarantee specific device availability times — windows change without notice
- NEVER recommend a device without checking its current status
- NEVER assume all QPU devices have equivalent capabilities
- NEVER claim a device is permanently unavailable based on current status

## Phase 3 — Rollback

1. If tasks were submitted to an alternative device, compare results with expected outcomes
2. Revert application code if device-specific changes were made
3. Document device availability observations for planning
4. Restore original device preferences in application configuration
5. Verify no tasks are stranded on devices that went offline

## Common Issues

- symptoms: "No QPU devices returned by search-devices"
  diagnosis: "Region may not have QPU devices, or filter is too restrictive."
  resolution: "Try different regions (us-east-1, us-west-1, eu-west-2). Remove filters to see all devices."

- symptoms: "Device shows RETIRED status"
  diagnosis: "QPU device has been permanently decommissioned by the provider."
  resolution: "Select an alternative device from the same or different provider. Update device ARN in application code."

## Output Format

```yaml
root_cause: "device_availability — <specific_cause>"
evidence:
  - type: device_status
    content: "<device statuses and availability info>"
severity: LOW
mitigation:
  immediate: "Select available alternative device"
  long_term: "Implement device selection strategy with fallbacks"
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
