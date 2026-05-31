---
title: "A1 — Device Provisioning Failures"
description: "Diagnose Panorama appliance provisioning and setup issues"
status: active
severity: HIGH
triggers:
  - "device provisioning failed"
  - "cannot register appliance"
  - "provisioning certificate error"
owner: devops-agent
objective: "Identify and resolve Panorama device provisioning failures"
context: "Provisioning requires physical access, Ethernet connectivity, USB drive with certificate, and outbound HTTPS to AWS endpoints."
---

## Phase 1 — Triage

MUST:
- Verify physical Ethernet connection to the appliance
- Check that the provisioning certificate was downloaded and placed on USB drive
- Verify network allows outbound HTTPS (443) to AWS endpoints
- Check device status: `aws panorama describe-device --device-id <device-id>`

SHOULD:
- Verify DNS resolution from the appliance network
- Check for proxy or firewall blocking AWS endpoints
- Verify the IAM role used for provisioning has panorama:* permissions
- Check CloudTrail for provisioning API errors

MAY:
- Test connectivity to iot.region.amazonaws.com from the network
- Verify NTP time sync on the network

## Phase 2 — Remediate

MUST:
- Ensure Ethernet cable is connected and link is active
- Re-download provisioning certificate if expired
- Open outbound HTTPS to required AWS endpoints

SHOULD:
- Configure DNS to resolve AWS endpoints
- Whitelist Panorama, IoT, and S3 endpoints in firewall
- Document network requirements for future provisioning

## Common Issues

- symptoms: "Device stuck in AWAITING_PROVISIONING"
  diagnosis: "Network cannot reach AWS endpoints."
  resolution: "Verify outbound HTTPS to panorama.region.amazonaws.com, iot.region.amazonaws.com."

- symptoms: "Certificate error during provisioning"
  diagnosis: "Provisioning certificate expired or corrupted."
  resolution: "Generate new provisioning certificate from Panorama console and re-provision."

## Output Format

```yaml
root_cause: "device_provisioning — <specific_cause>"
evidence:
  - type: device_status
    content: "<device provisioning status>"
  - type: network_connectivity
    content: "<network reachability results>"
severity: HIGH
mitigation:
  immediate: "Fix network/certificate issue and re-provision"
  long_term: "Document network requirements for provisioning"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — device provisioning involves certificates and credentials
- Service Disruption Risk: LOW — provisioning failures do not affect existing devices
- Compliance Impact: MEDIUM — device identity and authentication affect security posture

## Escalation Conditions
- Device provisioning fails repeatedly with certificate errors
- Device registration fails with network connectivity issues
- Provisioning credentials compromised
- Multiple devices fail to provision simultaneously
- Device identity verification fails

## Data Sensitivity
- Device certificates — RESTRICTED
- Provisioning credentials — RESTRICTED
- Device serial numbers — INTERNAL
- Network configuration details — INTERNAL
- Never expose device certificates or provisioning secrets

## Prohibited Actions
- Do NOT share device provisioning credentials
- Do NOT skip device identity verification during provisioning
- Do NOT provision devices on untrusted networks
- Do NOT reuse provisioning certificates across devices
- Do NOT bypass device registration security checks
- Do NOT store provisioning credentials in plaintext

## Phase 3 — Rollback
- Revoke certificates for incorrectly provisioned devices
- Delete failed device registrations
- Rotate provisioning credentials if compromised
- Restore previous provisioning configurations
- Verify device identity after re-provisioning
- Document provisioning failures and resolution steps

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
