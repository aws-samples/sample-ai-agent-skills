---
title: "D1 — MediaLive Integration"
description: "Diagnose Elemental Link to MediaLive integration issues"
status: active
severity: HIGH
triggers: ["MediaLive not receiving", "input not linked", "channel cannot see device", "integration error"]
owner: devops-agent
objective: "Resolve integration issues between Elemental Link devices and MediaLive channels"
context: "Elemental Link feeds video to MediaLive via an INPUT_DEVICE type input. The device must be in the same region as the MediaLive channel. The input must be created and attached to a channel."
---

## Phase 1 — Triage

MUST:
- List device inputs: `aws medialive list-inputs --query 'Inputs[?Type==`INPUT_DEVICE`].{Id:Id,Name:Name,State:State,AttachedChannels:AttachedChannels}'`
- Check device status: `aws medialive describe-input-device --input-device-id <device-id> --query '{ConnectionState:ConnectionState,Id:Id}'`
- Verify the input references the correct device: `aws medialive describe-input --input-id <input-id>`

SHOULD:
- Verify device and channel are in the same AWS region
- Check the MediaLive channel state: `aws medialive describe-channel --channel-id <channel-id> --query '{State:State,InputAttachments:InputAttachments}'`

MAY:
- Review CloudTrail for CreateInput or UpdateChannel events
- Check for IAM permission issues on the MediaLive service role

## Phase 2 — Remediate

MUST:
- Create an INPUT_DEVICE type input if one does not exist: `aws medialive create-input --name <name> --type INPUT_DEVICE --input-devices Id=<device-id>`
- Attach the input to the MediaLive channel (channel must be stopped)
- Verify the device is CONNECTED before starting the channel

SHOULD:
- Use a STANDARD class channel with two inputs for failover
- Verify the MediaLive service role has medialive:* permissions

MAY:
- Configure automatic input failover between two Elemental Link devices
- Set up CloudWatch alarms for channel input loss

## Safety Ratings

```
safety_ratings:
  - "list-inputs: GREEN — read-only enumeration"
  - "describe-input: GREEN — read-only input details"
  - "describe-channel: GREEN — read-only channel details"
  - "create-input: YELLOW — creates new MediaLive resource"
  - "update-channel: RED — modifies channel configuration, requires channel stop"
```

## Escalation Conditions

- Input creation fails with internal service error
- Channel cannot start despite correct input configuration
- Device shows CONNECTED but MediaLive reports no input signal
- Cross-region integration is required (device and channel in different regions)

## Data Sensitivity

- `aws medialive list-inputs`: LOW — returns input metadata
- `aws medialive describe-channel`: MEDIUM — reveals channel configuration including output destinations
- `aws medialive create-input`: MEDIUM — creates resources that may incur costs
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER modify a running channel's input attachments — stop the channel first
- NEVER delete an input that is attached to a channel
- NEVER assume cross-region device-to-channel connectivity works without transfer
- NEVER create duplicate inputs for the same device without understanding the use case

## Phase 3 — Rollback

1. If an input was created in error, detach it from any channel and delete: `aws medialive delete-input --input-id <input-id>`
2. If channel configuration was modified, restore previous input attachments (requires channel stop)
3. If the channel was stopped for troubleshooting, restart it: `aws medialive start-channel --channel-id <channel-id>`
4. Verify the stream is functioning correctly after any changes
5. Document integration changes for the operations team

## Common Issues

- symptoms: "Channel starts but shows 'No Input' alert"
  diagnosis: "Elemental Link device is DISCONNECTED or in a different region than the channel."
  resolution: "Verify device ConnectionState is CONNECTED and device region matches channel region."

## Output Format

```yaml
root_cause: "medialive_integration — <specific_cause>"
evidence:
  - type: integration_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix device-to-channel integration"
  long_term: "Implement integration monitoring and failover"
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
