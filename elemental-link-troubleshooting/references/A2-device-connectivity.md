---
title: "A2 — Device Connectivity"
description: "Diagnose Elemental Link device connectivity issues"
status: active
severity: HIGH
triggers: ["device disconnected", "connection lost", "device offline", "ConnectionState DISCONNECTED"]
owner: devops-agent
objective: "Restore connectivity between Elemental Link device and AWS"
context: "Elemental Link maintains a persistent connection to AWS. ConnectionState shows CONNECTED or DISCONNECTED. Intermittent connectivity causes stream interruptions."
---

## Phase 1 — Triage

MUST:
- Check device connection state: `aws medialive describe-input-device --input-device-id <device-id> --query '{ConnectionState:ConnectionState,DeviceSettingsSyncState:DeviceSettingsSyncState}'`
- Check CloudWatch for connectivity metrics: `aws cloudwatch get-metric-statistics --namespace AWS/MediaLive --metric-name NetworkIn --dimensions Name=InputDeviceId,Value=<device-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify physical network connection (LED indicators, cable integrity)

SHOULD:
- Check if other devices on the same network are also disconnected
- Review recent network changes (firewall rules, proxy configuration, ISP changes)

MAY:
- Run network diagnostics from a device on the same subnet to verify AWS endpoint reachability
- Check AWS Health Dashboard for MediaLive service issues

## Phase 2 — Remediate

MUST:
- Restore network path to AWS endpoints (outbound HTTPS port 443)
- Power cycle the device if network is confirmed working but device remains disconnected
- Verify DNS resolution is functioning on the device network

SHOULD:
- Ensure upload bandwidth meets minimum requirements (2x target bitrate)
- Check for network congestion or packet loss on the upload path

MAY:
- Configure QoS rules to prioritize Elemental Link traffic
- Set up CloudWatch alarms for device connectivity: `aws cloudwatch put-metric-alarm --alarm-name link-device-disconnect --namespace AWS/MediaLive --metric-name DeviceConnectionState --dimensions Name=InputDeviceId,Value=<device-id> --comparison-operator LessThanThreshold --threshold 1 --evaluation-periods 2 --period 60`

## Safety Ratings

```
safety_ratings:
  - "describe-input-device: GREEN — read-only device status check"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics retrieval"
  - "power cycle device: YELLOW — interrupts active stream"
  - "put-metric-alarm: YELLOW — creates monitoring resource, non-destructive"
```

## Escalation Conditions

- Device remains disconnected after network verification and power cycle
- Multiple devices disconnect simultaneously suggesting infrastructure issue
- Connectivity is intermittent with no identifiable network cause
- AWS Health Dashboard shows MediaLive service degradation

## Data Sensitivity

- `aws medialive describe-input-device`: LOW — returns device metadata and connection state
- `aws cloudwatch get-metric-statistics`: LOW — returns network throughput metrics
- Network diagnostic results: MEDIUM — may reveal internal network topology

## Prohibited Actions

- NEVER suggest SSH or console access to the Elemental Link device
- NEVER suggest modifying device firmware to fix connectivity
- NEVER open inbound ports to the device — Elemental Link uses outbound connections only
- NEVER suggest replacing the device before exhausting network troubleshooting

## Phase 3 — Rollback

1. If firewall rules were modified, revert to previous configuration if connectivity is not restored
2. If QoS rules were added and cause other network issues, remove them
3. If CloudWatch alarms were created in error, delete them: `aws cloudwatch delete-alarms --alarm-names link-device-disconnect`
4. If the device was power cycled during production, coordinate channel restart with the production team
5. Document the connectivity timeline and resolution for future reference

## Common Issues

- symptoms: "Device shows DISCONNECTED but network appears fine"
  diagnosis: "Corporate proxy or firewall intercepting HTTPS traffic to AWS endpoints."
  resolution: "Bypass proxy for AWS IP ranges or add firewall exceptions for MediaLive endpoints."

## Output Format

```yaml
root_cause: "device_connectivity — <specific_cause>"
evidence:
  - type: connectivity_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Restore device connectivity"
  long_term: "Implement connectivity monitoring and redundant network paths"
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
