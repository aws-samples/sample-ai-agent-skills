---
title: "E2 — Firmware and Software Version Issues"
description: "Diagnose firmware compatibility and version mismatch problems"
status: active
severity: MEDIUM
triggers:
  - "firmware version mismatch"
  - "incompatible software version"
  - "device needs update"
owner: devops-agent
objective: "Resolve firmware and software version issues"
context: "Panorama appliance firmware must be compatible with deployed applications and SDK versions."
---

## Phase 1 — Triage

MUST:
- Check current software version: `aws panorama describe-device --device-id <device-id> --query 'CurrentSoftware'`
- Compare with latest available version
- Check application compatibility requirements
- Review release notes for the target version

SHOULD:
- Verify all devices in fleet are on consistent versions
- Check for deprecation notices on current version

MAY:
- Test update on a non-production device first
- Review changelog for breaking changes

## Phase 2 — Remediate

MUST:
- Plan update to compatible version
- Verify application compatibility before updating
- Follow OTA update procedure (see E1)

SHOULD:
- Maintain version inventory across fleet
- Test applications against new firmware before fleet-wide rollout

## Common Issues

- symptoms: "Application fails after firmware update"
  diagnosis: "Application SDK version incompatible with new firmware."
  resolution: "Rebuild application with updated SDK. Or rollback firmware if possible."

## Output Format

```yaml
root_cause: "firmware — <specific_cause>"
evidence:
  - type: version_info
    content: "<current and target firmware versions>"
severity: MEDIUM
mitigation:
  immediate: "Update to compatible version"
  long_term: "Maintain version compatibility matrix"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — firmware issues are device-level concerns
- Service Disruption Risk: HIGH — firmware issues can render devices non-functional
- Compliance Impact: MEDIUM — outdated firmware may have security vulnerabilities

## Escalation Conditions
- Firmware corruption detected on device
- Device fails to boot after firmware change
- Firmware version incompatible with deployed applications
- Security vulnerability in current firmware version
- Firmware downgrade required but not supported

## Data Sensitivity
- Firmware versions and configurations — INTERNAL
- Device diagnostic logs — INTERNAL
- Boot logs — INTERNAL
- Hardware identifiers — INTERNAL
- Never expose firmware details that reveal device vulnerabilities

## Prohibited Actions
- Do NOT run devices with known firmware vulnerabilities
- Do NOT attempt manual firmware modifications
- Do NOT bypass firmware integrity checks
- Do NOT ignore firmware compatibility warnings
- Do NOT downgrade firmware without AWS guidance
- Do NOT deploy applications incompatible with current firmware

## Phase 3 — Rollback
- Restore previous firmware version if supported
- Contact AWS Support for firmware recovery procedures
- Migrate workloads to devices with compatible firmware
- Verify device functionality after firmware changes
- Document firmware issues and compatibility requirements
- Update firmware management procedures

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
