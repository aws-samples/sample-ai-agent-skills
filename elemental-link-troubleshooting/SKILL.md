---
name: elemental-link-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Elemental Link
  problems by analyzing device registration, connectivity, input
  configuration, video quality, network settings, HDMI/SDI inputs,
  MediaLive integration, firmware updates, monitoring, multi-device
  management, and following structured runbooks.
  Activate when: device registration failures, connectivity issues,
  input configuration errors, video quality degradation, network
  problems, HDMI/SDI input errors, MediaLive integration failures,
  channel attachment issues, firmware update problems, device
  monitoring gaps, multi-device management issues, or the user says
  something is wrong with Elemental Link.
compatibility: >
  Requires AWS CLI or SDK access with medialive:*, mediaconnect:*,
  cloudwatch:*, ec2:Describe*, iam:Get*, and logs:* permissions.
  Physical access to Elemental Link hardware may be required for
  HDMI/SDI and network troubleshooting.
---

# AWS Elemental Link Diagnostics

## When to use

Any AWS Elemental Link investigation — device registration, connectivity, input configuration, video quality, network configuration, HDMI/SDI inputs, MediaLive integration, channel attachment, firmware updates, device monitoring, or multi-device management.

## Investigation workflow

### Step 1 — Collect and triage

```
aws medialive list-input-devices --query 'InputDevices[*].{Id:Id,Name:Name,Type:Type,DeviceSettingsSyncState:DeviceSettingsSyncState,ConnectionState:ConnectionState}'
aws medialive list-inputs --query 'Inputs[*].{Id:Id,Name:Name,Type:Type,State:State,AttachedChannels:AttachedChannels}'
aws cloudwatch get-metric-statistics --namespace AWS/MediaLive --metric-name InputVideoFrameRate --dimensions Name=InputDeviceId,Value=<device-id> --start-time <start> --end-time <end> --period 300 --statistics Average
```

### Step 2 — Domain deep dive

```
aws medialive describe-input-device --input-device-id <device-id>
aws medialive describe-input --input-id <input-id>
aws medialive describe-input-device-thumbnail --input-device-id <device-id> --accept image/jpeg
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=medialive.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/MediaLive --metric-name NetworkIn --dimensions Name=InputDeviceId,Value=<device-id> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws medialive list-input-device-transfers --transfer-type INCOMING
```

Read `references/guardrails.md` before concluding on any Elemental Link issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `medialive list-input-devices` | List all Elemental Link devices |
| `medialive describe-input-device` | Get device details and status |
| `medialive describe-input-device-thumbnail` | Get current video thumbnail |
| `medialive list-inputs` | List MediaLive inputs |
| `medialive describe-input` | Get input configuration details |
| `medialive update-input-device` | Modify device settings |
| `medialive transfer-input-device` | Transfer device to another account |

## Gotchas: AWS Elemental Link

- Elemental Link is a HARDWARE encoder that ships to your location. It connects to AWS MediaLive via the internet — it is NOT a software-only service.
- Devices must be registered to an AWS account before use. Registration happens automatically when the device connects to the internet and phones home to AWS.
- Elemental Link supports HDMI and SDI input types depending on the hardware model (Link HD vs Link UHD). Not all models support both input types.
- The device requires a stable internet connection with sufficient upload bandwidth. AWS recommends at least 2x the target bitrate for reliable streaming.
- Device settings sync state indicates whether local device settings match the cloud configuration. SYNCED means they match; SYNCING means an update is in progress.
- Elemental Link devices are region-specific. A device registered in us-east-1 cannot be used with MediaLive channels in eu-west-1 without transfer.
- Firmware updates are managed by AWS and pushed automatically. You cannot manually install firmware — only approve or defer updates.

## Anti-hallucination rules

1. Always cite specific device IDs, input IDs, or API responses as evidence.
2. Elemental Link is physical hardware, NOT a software service. Never suggest software installation steps.
3. Device registration is automatic on first internet connection. Never suggest manual registration APIs that don't exist.
4. Firmware is managed by AWS. Never suggest downloading or manually installing firmware.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Registration | A1–A2 | Device registration failures, device connectivity |
| B — Input | B1–B2 | Input configuration errors, video quality issues |
| C — Network | C1–C2 | Network configuration, HDMI/SDI input errors |
| D — Integration | D1–D2 | MediaLive integration, channel attachment issues |
| E — Maintenance | E1–E2 | Device firmware updates, device monitoring |
| F — Fleet | F1 | Multi-device management |
| Z — Catch-All | Z1 | General troubleshooting |
