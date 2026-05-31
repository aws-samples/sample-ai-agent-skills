---
title: "E1 — OTA Update Failures"
description: "Diagnose over-the-air software update issues"
status: active
severity: HIGH
triggers:
  - "OTA update failed"
  - "update stuck"
  - "device update error"
owner: devops-agent
objective: "Resolve OTA update failures on Panorama appliances"
context: "OTA updates download and install new system software. Takes 30+ minutes. Must not be interrupted."
---

## Phase 1 — Triage

MUST:
- Check device status: `aws panorama describe-device --device-id <device-id>`
- Verify device has stable internet connectivity
- Check available software version: `aws panorama describe-device --device-id <device-id> --query 'LatestSoftware'`
- Review device logs in CloudWatch

SHOULD:
- Verify sufficient bandwidth for update download
- Check that no applications are in critical state during update
- Verify device is not mid-update before taking action

MAY:
- Check for known issues with the target software version
- Verify device storage has sufficient space

## Phase 2 — Remediate

MUST:
- Ensure stable network before retrying update
- Wait for any in-progress update to complete (do NOT interrupt)
- Contact AWS Support if device is stuck in update state

SHOULD:
- Schedule updates during maintenance windows
- Stop non-critical applications before updating

## Common Issues

- symptoms: "Update stuck in DOWNLOADING state"
  diagnosis: "Network bandwidth insufficient or connectivity intermittent."
  resolution: "Verify stable internet. Check bandwidth. Wait or retry with better connectivity."

- symptoms: "Device unresponsive after update"
  diagnosis: "Update may still be installing. Device reboots during update."
  resolution: "Wait 30-60 minutes. If still unresponsive, contact AWS Support."

## Output Format

```yaml
root_cause: "ota_update — <specific_cause>"
evidence:
  - type: update_status
    content: "<device update status>"
severity: HIGH
mitigation:
  immediate: "Restore connectivity and wait for update"
  long_term: "Plan maintenance windows for updates"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — OTA updates are firmware/software packages
- Service Disruption Risk: HIGH — failed updates can brick devices
- Compliance Impact: MEDIUM — updates may include security patches

## Escalation Conditions
- OTA update fails leaving device in degraded state
- Update causes device to become unresponsive
- Update rollback fails
- Critical security update cannot be applied
- Multiple devices fail update simultaneously

## Data Sensitivity
- Update packages — INTERNAL
- Device firmware versions — INTERNAL
- Update configurations — INTERNAL
- Update logs — INTERNAL
- Never expose update packages or firmware details

## Prohibited Actions
- Do NOT skip critical security updates
- Do NOT apply updates without testing on non-production devices first
- Do NOT update devices during critical monitoring periods without approval
- Do NOT ignore update failure notifications
- Do NOT apply updates from untrusted sources
- Do NOT bypass update validation procedures

## Phase 3 — Rollback
- Initiate device rollback to previous firmware version
- Contact AWS Support if device is unresponsive after update
- Restore previous device configurations
- Verify device functionality after rollback
- Re-schedule update after fixing identified issues
- Document update failures and device recovery steps

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
