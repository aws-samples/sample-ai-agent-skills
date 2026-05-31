---
title: "C2 — HDMI/SDI Input Errors"
description: "Diagnose Elemental Link HDMI and SDI input signal errors"
status: active
severity: MEDIUM
triggers: ["no video signal", "HDMI not detected", "SDI no signal", "black screen", "input error"]
owner: devops-agent
objective: "Resolve physical video input issues on Elemental Link devices"
context: "Elemental Link HD supports HDMI only. Link UHD supports HDMI and SDI. Physical cable connections, signal format, and HDCP can cause input failures."
---

## Phase 1 — Triage

MUST:
- Check device and input status: `aws medialive describe-input-device --input-device-id <device-id>`
- Verify video signal via thumbnail: `aws medialive describe-input-device-thumbnail --input-device-id <device-id> --accept image/jpeg`
- Confirm the device model supports the input type (HD = HDMI only, UHD = HDMI + SDI)

SHOULD:
- Check physical cable connections and verify cable integrity
- Verify the source device is outputting a supported resolution and frame rate
- Check LED indicators on the Elemental Link device for input signal status

MAY:
- Test with a different video source to isolate the issue
- Test with a different cable to rule out cable failure

## Phase 2 — Remediate

MUST:
- Ensure the correct input type is selected in device configuration: `aws medialive update-input-device --input-device-id <device-id> --hd-device-settings '{"ConfiguredInput":"HDMI"}'`
- Verify the source device is powered on and outputting video
- Check that HDCP is disabled on the source device (Elemental Link does not support HDCP-protected content)

SHOULD:
- Use high-quality cables rated for the signal type (HDMI 2.0 for 4K, 3G-SDI for HD)
- Keep cable runs within specification limits (HDMI: 15m max, SDI: varies by standard)

MAY:
- Use an HDMI signal analyzer to verify source output format
- Insert an HDMI splitter/repeater for long cable runs

## Safety Ratings

```
safety_ratings:
  - "describe-input-device: GREEN — read-only device status check"
  - "describe-input-device-thumbnail: GREEN — read-only thumbnail retrieval"
  - "update-input-device ConfiguredInput: YELLOW — changes active input source, may interrupt stream"
  - "physical cable changes: YELLOW — interrupts video signal during swap"
```

## Escalation Conditions

- Device reports no input signal with confirmed working source and cable
- Thumbnail shows corrupted video that does not match the source
- Device accepts signal intermittently suggesting hardware failure
- Multiple input ports on UHD device fail simultaneously

## Data Sensitivity

- `aws medialive describe-input-device`: LOW — returns device configuration and input status
- `aws medialive describe-input-device-thumbnail`: LOW — returns single video frame
- Physical device inspection: LOW — visual check of cables and LEDs

## Prohibited Actions

- NEVER suggest SDI troubleshooting for Elemental Link HD devices — they have no SDI port
- NEVER suggest HDCP bypass or circumvention tools
- NEVER open the Elemental Link device enclosure — this voids the warranty
- NEVER suggest firmware modification to add input support that the hardware does not have

## Phase 3 — Rollback

1. Revert ConfiguredInput setting if the change did not resolve the issue: `aws medialive update-input-device --input-device-id <device-id> --hd-device-settings '{"ConfiguredInput":"<previous-setting>"}'`
2. Reconnect the original cable if a replacement cable was tested
3. Reconnect the original video source if a test source was used
4. Verify the stream is functioning correctly after any physical changes
5. Document the input issue and resolution for the production team

## Common Issues

- symptoms: "Thumbnail shows black frame despite source device being on"
  diagnosis: "HDCP enabled on source device — Elemental Link cannot process protected content."
  resolution: "Disable HDCP on the source device output settings."

## Output Format

```yaml
root_cause: "hdmi_sdi_input — <specific_cause>"
evidence:
  - type: input_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix physical input connection or configuration"
  long_term: "Document supported input formats and cable requirements"
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
