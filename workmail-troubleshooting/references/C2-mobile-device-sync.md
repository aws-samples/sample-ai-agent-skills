---
title: "C2 — Mobile Device Sync"
description: "Diagnose WorkMail mobile device synchronization issues"
status: active
severity: MEDIUM
triggers: ["mobile not syncing", "phone email broken", "ActiveSync error", "calendar not syncing"]
owner: devops-agent
objective: "Resolve mobile device sync issues including EAS configuration, device policies, and sync errors"
context: "WorkMail uses Exchange ActiveSync (EAS) for mobile devices. EAS provides email, calendar, and contacts sync."
---

## Phase 1 — Triage

MUST:
- Verify user account is enabled: `aws workmail describe-user --organization-id <org-id> --user-id <user-id>`
- Check mobile device policies: `aws workmail list-mobile-device-access-rules --organization-id <org-id>`
- Verify EAS settings on the device: server `mobile.mail.<region>.awsapps.com`
- Check if the device type is allowed by mobile device access rules

SHOULD:
- Check access control rules for mobile protocol restrictions: `aws workmail list-access-control-rules --organization-id <org-id>`
- Verify the device OS version is supported
- Test with the web client to confirm account is functional

MAY:
- Check if the device was previously wiped or blocked
- Review device sync logs on the mobile device
- Test with a different mobile device to isolate device-specific issues

## Phase 2 — Remediate

MUST:
- Configure correct EAS settings: server `mobile.mail.<region>.awsapps.com`, SSL enabled
- Allow the device type in mobile device access rules if blocked
- Reset user password if authentication fails on the device

SHOULD:
- Remove and re-add the email account on the device
- Update device OS to the latest supported version

MAY:
- Configure mobile device management (MDM) policies
- Set up device wipe capability for lost devices

## Safety Ratings

```
safety_ratings:
  - "workmail describe-user: GREEN — read-only user inspection"
  - "workmail list-mobile-device-access-rules: GREEN — read-only rule listing"
  - "workmail create-mobile-device-access-rule: YELLOW — modifies device access policy"
  - "workmail delete-mobile-device-access-override: YELLOW — removes device-specific override"
```

## Escalation Conditions

- Mobile sync fails for all users suggesting service-wide EAS issue
- Specific device model consistently fails despite correct configuration
- Mobile device access rules cannot be modified due to compliance requirements
- Calendar or contacts sync fails while email sync works

## Data Sensitivity

- `aws workmail describe-user`: MEDIUM — reveals user details
- `aws workmail list-mobile-device-access-rules`: MEDIUM — reveals device policies
- Mobile device logs: MEDIUM — may contain email metadata
- Device wipe capability: HIGH — can erase all data on the device

## Prohibited Actions

- NEVER initiate a remote device wipe without explicit user approval
- NEVER disable mobile device access rules organization-wide
- NEVER configure EAS without SSL/TLS encryption
- NEVER share user credentials for testing on a different device

## Phase 3 — Rollback

1. If mobile device access rules were changed, restore original rules
2. If the email account was removed from the device, re-add with correct settings
3. If a device override was deleted, recreate it if needed
4. Verify email, calendar, and contacts all sync correctly after changes
5. Document the device configuration for the user's reference

## Common Issues

- symptoms: "Mobile device cannot connect to WorkMail"
  diagnosis: "EAS settings incorrect or device type blocked by access rules."
  resolution: "Use EAS server: mobile.mail.<region>.awsapps.com with SSL. Check device access rules."

- symptoms: "Email syncs but calendar does not"
  diagnosis: "Calendar sync may be disabled in the device's account settings."
  resolution: "Check device account settings. Enable calendar sync. Remove and re-add account if needed."

## Output Format

```yaml
root_cause: "mobile_sync — <specific_cause>"
evidence:
  - type: sync_error
    content: "<device settings, access rules, sync status>"
severity: MEDIUM
mitigation:
  immediate: "Fix device configuration or access rules"
  long_term: "Document mobile setup procedures for users"
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
