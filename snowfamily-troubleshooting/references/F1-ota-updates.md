---
title: "F1 — OTA Updates"
description: "Diagnose Snowball Edge over-the-air update issues"
status: active
severity: MEDIUM
triggers: ["OTA update failed", "software update error", "update not available", "update stuck"]
owner: devops-agent
objective: "Resolve OTA update issues including connectivity, update availability, and installation failures"
context: "OTA updates require internet connectivity to AWS endpoints. Updates include security patches, bug fixes, and feature updates."
---

## Phase 1 — Triage

MUST:
- Check device software version: `snowballEdge describe-device --endpoint https://<device-ip>:8243 --manifest-file <manifest> --unlock-code <code>`
- Check for available updates: `snowballEdge check-for-updates --endpoint https://<device-ip>:8243 --manifest-file <manifest> --unlock-code <code>`
- Verify device has internet connectivity to AWS endpoints

SHOULD:
- Check DNS resolution for AWS endpoints
- Verify no proxy or firewall blocks HTTPS to AWS

MAY:
- Review update history on the device
- Check AWS announcements for known update issues

## Phase 2 — Remediate

MUST:
- Ensure device has internet connectivity (HTTPS to AWS endpoints)
- Configure DNS resolution for *.amazonaws.com
- Allow outbound HTTPS (443) to AWS update endpoints in firewall

SHOULD:
- Schedule updates during maintenance windows (updates may restart services)
- Back up critical data before applying updates

MAY:
- Configure proxy settings if device is behind a corporate proxy

## Safety Ratings

```
safety_ratings:
  - "describe-device: GREEN — read-only device status"
  - "check-for-updates: GREEN — read-only update check"
  - "download-updates: YELLOW — downloads update packages to device"
  - "install-updates: RED — installs updates, may restart services"
```

## Escalation Conditions

- Update installation fails and device becomes unresponsive
- Update requires device reboot during active data transfer
- Device is in an air-gapped environment and cannot reach AWS
- Update introduces compatibility issues with running workloads

## Data Sensitivity

- `snowballEdge describe-device`: MEDIUM — reveals device software version
- `snowballEdge check-for-updates`: LOW — reveals available updates
- Network connectivity test: LOW — reveals network configuration

## Prohibited Actions

- NEVER install updates during active data transfers
- NEVER interrupt an update installation in progress
- NEVER apply updates to production devices without testing on a non-production device first
- NEVER assume updates are backward compatible — check release notes

## Phase 3 — Rollback

1. If an update caused issues, contact AWS Support for rollback options
2. If services stopped after update, restart them manually
3. Verify all services are functional after update completion
4. Check data integrity after update-related service restarts
5. Document the update version and any issues encountered

## Common Issues

- symptoms: "check-for-updates returns 'Unable to connect to update service'"
  diagnosis: "Device does not have internet connectivity or DNS is not resolving AWS endpoints."
  resolution: "Configure internet access. Verify DNS resolves *.amazonaws.com. Allow outbound HTTPS to AWS."

## Output Format

```yaml
root_cause: "ota_update — <specific_cause>"
evidence:
  - type: update_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix update connectivity"
  long_term: "Schedule regular updates"
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
