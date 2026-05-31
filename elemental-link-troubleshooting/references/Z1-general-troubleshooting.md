---
title: "Z1 — General Elemental Link Troubleshooting"
description: "Catch-all runbook for Elemental Link issues"
status: active
severity: MEDIUM
triggers: ["Elemental Link error", "Link device not working", "Link issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified Elemental Link issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List all devices: `aws medialive list-input-devices --query 'InputDevices[*].{Id:Id,Name:Name,ConnectionState:ConnectionState,DeviceSettingsSyncState:DeviceSettingsSyncState}'`
- Check device details: `aws medialive describe-input-device --input-device-id <device-id>`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=medialive.amazonaws.com --max-results 20`

SHOULD:
- Check associated MediaLive inputs and channels
- Verify CloudWatch metrics are being published

MAY:
- Check AWS Health Dashboard for MediaLive service issues
- Review recent configuration changes across the fleet

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook (A1-F1)
- Document findings from initial triage

SHOULD:
- Classify the issue by category (registration, connectivity, input, integration, firmware, fleet)
- Collect relevant evidence before escalating

MAY:
- Set up temporary monitoring for the specific issue
- Contact AWS Support with collected evidence if no runbook matches

## Safety Ratings

```
safety_ratings:
  - "list-input-devices: GREEN — read-only enumeration"
  - "describe-input-device: GREEN — read-only device details"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "check AWS Health Dashboard: GREEN — read-only service status"
```

## Escalation Conditions

- Issue does not match any specific runbook after systematic investigation
- Multiple devices affected simultaneously suggesting service-level issue
- Device hardware appears physically damaged or non-functional
- Issue requires AWS internal investigation (device registration backend)

## Data Sensitivity

- `aws medialive list-input-devices`: LOW — returns device metadata
- `aws medialive describe-input-device`: LOW — returns device configuration
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER make configuration changes during general troubleshooting without identifying root cause
- NEVER suggest the device is defective without exhausting software and network troubleshooting
- NEVER transfer or delete devices as part of general troubleshooting
- NEVER dismiss the issue without documenting the investigation path

## Phase 3 — Rollback

1. Revert any configuration changes made during investigation
2. Restore device settings if they were modified during troubleshooting
3. If a MediaLive channel was stopped for investigation, restart it after confirming the issue
4. Document all investigation steps and findings
5. Verify all devices and channels are in their expected state before closing

## Common Issues

- symptoms: "Device was working yesterday but not today"
  diagnosis: "Check for network changes, firmware updates, or account-level changes overnight."
  resolution: "Follow triage steps to identify the specific domain, then pivot to the appropriate runbook."

## Output Format

```yaml
root_cause: "general_elemental_link — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring and documentation"
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
