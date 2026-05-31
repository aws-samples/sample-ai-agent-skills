---
title: "A2 — Device Registration Errors"
description: "Diagnose device registration and identity issues"
status: active
severity: HIGH
triggers:
  - "device registration failed"
  - "device not appearing in console"
  - "device identity error"
owner: devops-agent
objective: "Resolve device registration and identity problems"
context: "Device registration creates an IoT thing and certificate. Registration failures prevent device management."
---

## Phase 1 — Triage

MUST:
- List devices: `aws panorama list-devices`
- Check device status: `aws panorama describe-device --device-id <device-id>`
- Verify IAM permissions for panorama:ProvisionDevice
- Check CloudTrail for registration errors

SHOULD:
- Verify the device serial number matches the expected appliance
- Check IoT Core for the device thing: `aws iot describe-thing --thing-name <device-id>`
- Verify device certificate status in IoT Core

MAY:
- Check for duplicate device registrations
- Verify account limits for Panorama devices

## Phase 2 — Remediate

MUST:
- Re-register device if registration was incomplete
- Ensure IAM role has required permissions
- Verify device connectivity to IoT Core endpoint

SHOULD:
- Deregister and re-register if device is in error state
- Update device description for tracking

## Common Issues

- symptoms: "Device shows ERROR status after registration"
  diagnosis: "Device cannot communicate with IoT Core."
  resolution: "Verify network connectivity and IoT endpoint access. Re-provision if needed."

## Output Format

```yaml
root_cause: "registration — <specific_cause>"
evidence:
  - type: device_status
    content: "<device registration status>"
severity: HIGH
mitigation:
  immediate: "Fix registration issue"
  long_term: "Automate device registration validation"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — registration errors are operational
- Service Disruption Risk: LOW — registration failures do not affect running devices
- Compliance Impact: MEDIUM — unregistered devices cannot be managed

## Escalation Conditions
- Registration fails with internal service errors
- Registration blocked by IAM permission issues
- Device cannot reach Panorama service endpoints
- Registration certificate validation fails
- Multiple devices fail registration simultaneously

## Data Sensitivity
- Registration tokens — RESTRICTED
- Device identifiers — INTERNAL
- IAM role configurations — INTERNAL
- Network endpoint details — INTERNAL
- Never expose registration tokens externally

## Prohibited Actions
- Do NOT share registration tokens between devices
- Do NOT bypass registration validation steps
- Do NOT register devices without proper network security
- Do NOT ignore registration failure patterns
- Do NOT use expired registration tokens
- Do NOT register devices to unauthorized accounts

## Phase 3 — Rollback
- Revoke failed registration tokens
- Clean up partially registered device entries
- Restore IAM permissions if modified
- Re-generate registration tokens for retry
- Verify device registration status after retry
- Document registration issues and resolutions

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
