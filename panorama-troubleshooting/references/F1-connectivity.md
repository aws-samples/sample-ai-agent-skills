---
title: "F1 — Network Connectivity Issues"
description: "Diagnose device-to-cloud connectivity problems"
status: active
severity: HIGH
triggers:
  - "device offline"
  - "cannot reach AWS"
  - "connectivity lost"
owner: devops-agent
objective: "Resolve network connectivity between Panorama appliance and AWS"
context: "Panorama appliance requires outbound HTTPS (443) to multiple AWS endpoints including IoT, S3, CloudWatch, and Panorama service."
---

## Phase 1 — Triage

MUST:
- Check device status: `aws panorama describe-device --device-id <device-id>`
- Verify physical Ethernet connection
- Check if device appears as ONLINE or OFFLINE
- Verify outbound HTTPS (443) is allowed to AWS endpoints

SHOULD:
- Check DNS resolution from the device network
- Verify no firewall changes were made recently
- Check for network outages in the device's location
- Test connectivity to: panorama.region.amazonaws.com, iot.region.amazonaws.com

MAY:
- Check DHCP lease and IP assignment
- Verify no MAC address filtering is blocking the device

## Phase 2 — Remediate

MUST:
- Restore physical network connection
- Open outbound HTTPS to required AWS endpoints
- Verify DNS resolution

SHOULD:
- Document all required AWS endpoints for firewall rules
- Set up network monitoring for the appliance

## Common Issues

- symptoms: "Device shows OFFLINE in console"
  diagnosis: "Network connectivity to AWS endpoints lost."
  resolution: "Check Ethernet, DNS, firewall rules. Verify outbound HTTPS to AWS."

## Output Format

```yaml
root_cause: "connectivity — <specific_cause>"
evidence:
  - type: device_status
    content: "<device online/offline status>"
severity: HIGH
mitigation:
  immediate: "Restore network connectivity"
  long_term: "Monitor network health and document requirements"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — connectivity issues prevent data flow
- Service Disruption Risk: HIGH — connectivity loss stops all device operations
- Compliance Impact: HIGH — device connectivity is required for management and monitoring

## Escalation Conditions
- Device loses all network connectivity
- Connectivity issues affect multiple devices simultaneously
- DNS resolution failures prevent cloud communication
- Firewall changes block device traffic
- Device cannot reach required AWS service endpoints

## Data Sensitivity
- Network configurations — INTERNAL
- DNS settings — INTERNAL
- Firewall rules — INTERNAL
- Device IP addresses — INTERNAL
- Never expose device network topology externally

## Prohibited Actions
- Do NOT expose device management ports to the public internet
- Do NOT bypass network security controls for troubleshooting
- Do NOT modify firewall rules without understanding device traffic patterns
- Do NOT ignore persistent connectivity issues
- Do NOT disable device network monitoring
- Do NOT route device traffic over untrusted networks

## Phase 3 — Rollback
- Restore previous network configurations
- Revert firewall rule changes
- Restore DNS configurations
- Verify device connectivity after changes
- Re-enable network monitoring
- Document connectivity issues and network requirements

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
