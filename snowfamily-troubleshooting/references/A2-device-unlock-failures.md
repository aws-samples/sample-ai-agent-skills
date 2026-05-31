---
title: "A2 — Device Unlock Failures"
description: "Diagnose AWS Snow Family device unlock and initialization failures"
status: active
severity: CRITICAL
triggers: ["cannot unlock device", "unlock failed", "manifest error", "unlock code invalid"]
owner: devops-agent
objective: "Resolve device unlock issues including manifest download, unlock code, and network connectivity"
context: "Unlocking requires the Snowball Edge client, correct manifest file, unlock code, and network access to the device on port 8243."
---

## Phase 1 — Triage

MUST:
- Verify job status is InProgress: `aws snowball describe-job --job-id <job-id> --query 'JobMetadata.JobState'`
- Download manifest: `aws snowball get-job-manifest --job-id <job-id>`
- Get unlock code: `aws snowball get-job-unlock-code --job-id <job-id>`

SHOULD:
- Verify network connectivity to device IP on port 8243
- Confirm Snowball Edge client is installed and up to date

MAY:
- Check device LCD display for IP address and status
- Test network path: `ping <device-ip>` and `curl -k https://<device-ip>:8243`

## Phase 2 — Remediate

MUST:
- Use correct manifest file for the specific job (not a different job's manifest)
- Use correct unlock code for the specific job
- Ensure network connectivity to device IP on port 8243

SHOULD:
- Configure device network via LCD display if IP is not reachable
- Update Snowball Edge client to latest version

MAY:
- Use a direct Ethernet connection if network routing is problematic

## Safety Ratings

```
safety_ratings:
  - "describe-job: GREEN — read-only job details"
  - "get-job-manifest: YELLOW — downloads cryptographic manifest file"
  - "get-job-unlock-code: YELLOW — retrieves sensitive unlock code"
  - "snowballEdge unlock-device: YELLOW — unlocks device for data access"
```

## Escalation Conditions

- Device does not power on or LCD display is non-functional
- Manifest download fails with access denied after verifying IAM permissions
- Device reports tamper detection or hardware error on LCD
- Multiple unlock attempts fail with correct manifest and unlock code

## Data Sensitivity

- `aws snowball get-job-manifest`: HIGH — cryptographic manifest for device access
- `aws snowball get-job-unlock-code`: HIGH — unlock code for device access
- `snowballEdge describe-device`: MEDIUM — reveals device configuration and status

## Prohibited Actions

- NEVER share manifest files or unlock codes via unencrypted channels
- NEVER attempt to physically open or modify the Snow device
- NEVER use one job's manifest to unlock a different job's device
- NEVER store manifest files and unlock codes in the same location

## Phase 3 — Rollback

1. If the device was unlocked with incorrect configuration, lock it: `snowballEdge lock-device`
2. If network configuration was changed on the device, restore original settings via LCD
3. Securely delete downloaded manifest files after use
4. If the device is unresponsive after unlock attempts, contact AWS Support
5. Document the unlock procedure and network configuration for the team

## Common Issues

- symptoms: "snowballEdge unlock-device returns 'Invalid manifest or unlock code'"
  diagnosis: "Manifest file is from a different job or unlock code is incorrect."
  resolution: "Re-download manifest and unlock code for the correct job ID. Verify job status is InProgress."

## Output Format

```yaml
root_cause: "device_unlock — <specific_cause>"
evidence:
  - type: unlock_error
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix unlock issue"
  long_term: "Document unlock procedures"
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
