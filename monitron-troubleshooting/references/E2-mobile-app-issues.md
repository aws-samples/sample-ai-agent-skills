---
title: "E2 — Mobile App Issues"
description: "Diagnose Monitron mobile app login, functionality, and device interaction issues"
status: active
severity: MEDIUM
triggers: ["app not working", "cannot login", "app crash", "app error", "mobile app"]
owner: devops-agent
objective: "Resolve Monitron mobile app issues for device management and monitoring"
context: "The Monitron mobile app is required for gateway commissioning, sensor pairing, and on-site monitoring. It uses Bluetooth for device communication and IAM Identity Center for authentication."
---

## Phase 1 — Triage

MUST:
- Verify the app is updated to the latest version (check app store)
- Confirm the user has valid IAM Identity Center credentials
- Check if Bluetooth is enabled on the mobile device

SHOULD:
- Verify the mobile device OS meets minimum requirements (iOS 14+ or Android 8+)
- Check if the user has project access: `aws monitron list-project-admin-users --project-id <project-id>`
- Test login with a different user account to isolate user vs. app issue

MAY:
- Clear app cache and data (Android) or reinstall (iOS)
- Check if the mobile device has location services enabled (required for BLE)

## Phase 2 — Remediate

MUST:
- Update the Monitron mobile app to the latest version
- Ensure Bluetooth and location services are enabled on the mobile device
- Verify IAM Identity Center credentials are valid and not expired

SHOULD:
- Grant all required app permissions (Bluetooth, location, camera for QR codes)
- Ensure the mobile device has internet connectivity

MAY:
- Reinstall the app if persistent issues after cache clear
- Try a different mobile device to isolate device-specific issues

## Common Issues

- symptoms: "App login fails with authentication error"
  diagnosis: "IAM Identity Center credentials expired or user not provisioned."
  resolution: "Reset user password in IAM Identity Center. Verify user is provisioned and active."

- symptoms: "App cannot find gateway during commissioning"
  diagnosis: "Bluetooth disabled, location services off, or gateway not in pairing mode."
  resolution: "Enable Bluetooth and location services. Power cycle gateway. Ensure physical proximity."

## Safety Ratings

```
safety_ratings:
  - "App update: GREEN — standard app maintenance"
  - "App reinstall: YELLOW — may lose cached data and preferences"
  - "Password reset: YELLOW — affects user access temporarily"
  - "monitron list-project-admin-users: GREEN — read-only user listing"
```

## Escalation Conditions

- App crashes consistently on specific mobile device models
- Authentication issues affect multiple users simultaneously
- App cannot communicate with gateways despite Bluetooth being enabled
- App version is incompatible with current Monitron service version

## Data Sensitivity

- User login credentials: HIGH — authentication information
- App diagnostic logs: MEDIUM — may contain device and user information
- Bluetooth device scan results: LOW — nearby device identifiers
- `aws monitron list-project-admin-users`: MEDIUM — reveals admin identities

## Prohibited Actions

- NEVER share user login credentials for troubleshooting
- NEVER install the app from unofficial sources (sideloading)
- NEVER disable mobile device security to troubleshoot app issues
- NEVER use rooted/jailbroken devices for Monitron app

## Phase 3 — Rollback

1. If app was reinstalled, log in again with IAM Identity Center credentials
2. If user password was reset, communicate new temporary password securely
3. If app permissions were changed, restore required permissions
4. Verify app functionality after any changes
5. Document the app issue for the support team

## Output Format

```yaml
root_cause: "mobile_app — <specific_cause>"
evidence:
  - type: app_error
    content: "<app version, OS version, error message>"
severity: MEDIUM
mitigation:
  immediate: "Update app and verify permissions"
  long_term: "Establish mobile device requirements for Monitron users"
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
