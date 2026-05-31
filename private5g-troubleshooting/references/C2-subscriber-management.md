---
title: "C2 — Subscriber Management"
description: "Diagnose subscriber management and device identity issues"
status: active
severity: MEDIUM
triggers: ["subscriber error", "device identity issue", "too many devices", "subscriber limit"]
owner: devops-agent
objective: "Resolve subscriber management and device identity configuration issues"
context: "Subscribers are managed through device identifiers (SIMs). Each SIM represents one subscriber on the network."
---

## Phase 1 — Triage

MUST:
- List all device identifiers: `aws private-networks list-device-identifiers --network-arn <arn>`
- Check subscriber count against network limits
- Verify device identifier statuses: `aws private-networks get-device-identifier --device-identifier-arn <arn>`

SHOULD:
- Identify inactive or unused SIMs that can be deactivated
- Review subscriber distribution across network sites
- Check CloudTrail for subscriber management events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DeactivateDeviceIdentifier --max-results 10`

MAY:
- Export subscriber list for inventory reconciliation
- Check service quotas for device identifier limits

## Phase 2 — Remediate

MUST:
- Deactivate SIMs for devices no longer in use
- Ensure each active SIM corresponds to a legitimate device
- Verify subscriber count is within service quotas

SHOULD:
- Implement a subscriber lifecycle management process
- Tag device identifiers for tracking and organization

MAY:
- Request quota increase if legitimate subscriber count exceeds limits
- Automate subscriber provisioning with API integration

## Safety Ratings

```
safety_ratings:
  - "list-device-identifiers: GREEN — read-only enumeration"
  - "get-device-identifier: GREEN — read-only subscriber details"
  - "deactivate-device-identifier: YELLOW — removes device network access"
  - "service-quotas get-service-quota: GREEN — read-only quota check"
```

## Escalation Conditions

- Subscriber count approaching or exceeding service quota limits
- Unauthorized device identifiers found on the network
- Bulk subscriber provisioning failures
- Subscriber data inconsistency between API and actual devices

## Data Sensitivity

- `aws private-networks list-device-identifiers`: MEDIUM — reveals all subscriber identities
- Subscriber inventory: HIGH — contains IMSI/ICCID for all network users
- Device-to-subscriber mapping: MEDIUM — reveals which devices are on the network

## Prohibited Actions

- NEVER bulk-deactivate subscribers without explicit approval
- NEVER share subscriber lists or IMSI/ICCID data externally
- NEVER activate device identifiers without verifying the physical SIM exists
- NEVER modify subscriber configurations during peak usage hours without a maintenance window

## Phase 3 — Rollback

1. If subscribers were deactivated in error, reactivate them: `aws private-networks activate-device-identifier --device-identifier-arn <arn>`
2. Verify all legitimate devices can reconnect after subscriber changes
3. Restore any subscriber tags or metadata that were modified
4. Confirm subscriber count matches expected inventory
5. Notify affected users if their connectivity was interrupted

## Common Issues

- symptoms: "Cannot activate more device identifiers — quota exceeded"
  diagnosis: "Network has reached the maximum number of device identifiers."
  resolution: "Deactivate unused SIMs or request a service quota increase."

- symptoms: "Device identifier shows INACTIVE but SIM is physically in use"
  diagnosis: "SIM was deactivated via API but not physically removed from device."
  resolution: "Reactivate the device identifier if the SIM should be active."

## Output Format

```yaml
root_cause: "subscriber_management — <specific_cause>"
evidence:
  - type: subscriber_status
    content: "<status details>"
severity: MEDIUM
mitigation:
  immediate: "Resolve subscriber issue"
  long_term: "Implement subscriber lifecycle management"
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
