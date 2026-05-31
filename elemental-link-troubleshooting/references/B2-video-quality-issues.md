---
title: "B2 — Video Quality Issues"
description: "Diagnose Elemental Link video quality degradation"
status: active
severity: MEDIUM
triggers: ["poor video quality", "pixelation", "frame drops", "bitrate too low", "blurry stream"]
owner: devops-agent
objective: "Identify and resolve video quality issues originating from Elemental Link encoding"
context: "Video quality depends on source signal quality, encoding settings (bitrate, resolution), and network bandwidth. Elemental Link encodes locally and streams to MediaLive."
---

## Phase 1 — Triage

MUST:
- Check current encoding metrics: `aws cloudwatch get-metric-statistics --namespace AWS/MediaLive --metric-name InputVideoFrameRate --dimensions Name=InputDeviceId,Value=<device-id> --start-time <start> --end-time <end> --period 60 --statistics Average`
- Verify device configuration and bitrate: `aws medialive describe-input-device --input-device-id <device-id>`
- Check thumbnail for visual quality: `aws medialive describe-input-device-thumbnail --input-device-id <device-id> --accept image/jpeg`

SHOULD:
- Check network throughput metrics: `aws cloudwatch get-metric-statistics --namespace AWS/MediaLive --metric-name NetworkIn --dimensions Name=InputDeviceId,Value=<device-id> --start-time <start> --end-time <end> --period 60 --statistics Average`
- Verify the source video signal is clean (no issues at the camera or source device)

MAY:
- Compare current metrics against baseline from a known-good period
- Check for network congestion during quality degradation windows

## Phase 2 — Remediate

MUST:
- Ensure MaxBitrate is set appropriately for the content type and resolution
- Verify upload bandwidth is at least 2x the configured MaxBitrate
- Check that the source video signal is stable and at the expected resolution

SHOULD:
- Increase MaxBitrate if bandwidth allows: `aws medialive update-input-device --input-device-id <device-id> --hd-device-settings '{"MaxBitrate":<higher-bitrate>}'`
- Reduce encoding resolution if bandwidth is constrained

MAY:
- Implement network QoS to prioritize Elemental Link traffic
- Set up CloudWatch alarms for frame rate drops

## Safety Ratings

```
safety_ratings:
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics retrieval"
  - "describe-input-device: GREEN — read-only device status check"
  - "describe-input-device-thumbnail: GREEN — read-only thumbnail retrieval"
  - "update-input-device: YELLOW — modifies encoding settings, affects active stream quality"
```

## Escalation Conditions

- Quality degradation persists despite adequate bandwidth and correct settings
- Frame rate drops below acceptable threshold with a stable source signal
- Device thumbnail shows artifacts not present in the source video
- Quality issues correlate with specific times suggesting external interference

## Data Sensitivity

- `aws cloudwatch get-metric-statistics`: LOW — returns encoding performance metrics
- `aws medialive describe-input-device`: LOW — returns device configuration
- `aws medialive describe-input-device-thumbnail`: LOW — returns single video frame

## Prohibited Actions

- NEVER claim Elemental Link provides VMAF, MOS, or perceptual quality scores
- NEVER suggest replacing the device before investigating encoding settings and network
- NEVER set MaxBitrate beyond the available upload bandwidth
- NEVER modify encoding settings during live production without coordination

## Phase 3 — Rollback

1. Revert bitrate settings if changes worsened quality: `aws medialive update-input-device --input-device-id <device-id> --hd-device-settings '{"MaxBitrate":<original-bitrate>}'`
2. Remove QoS rules if they caused other network issues
3. Delete CloudWatch alarms if they were misconfigured: `aws cloudwatch delete-alarms --alarm-names <alarm-name>`
4. Document the quality issue timeline and metrics for post-incident review
5. Verify stream quality has returned to acceptable levels before closing

## Common Issues

- symptoms: "Video is pixelated despite high bitrate setting"
  diagnosis: "Upload bandwidth insufficient — actual throughput below configured bitrate."
  resolution: "Reduce MaxBitrate to match available bandwidth or increase upload bandwidth."

## Output Format

```yaml
root_cause: "video_quality — <specific_cause>"
evidence:
  - type: quality_metrics
    content: "<metric details>"
severity: MEDIUM
mitigation:
  immediate: "Adjust encoding settings or resolve bandwidth issue"
  long_term: "Implement bandwidth monitoring and quality alerting"
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
