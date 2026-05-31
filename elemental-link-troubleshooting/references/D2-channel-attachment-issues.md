---
title: "D2 — Channel Attachment Issues"
description: "Diagnose Elemental Link channel attachment and detachment issues"
status: active
severity: MEDIUM
triggers: ["cannot attach input", "channel attachment failed", "input already attached", "detach input error"]
owner: devops-agent
objective: "Resolve issues attaching or detaching Elemental Link inputs to MediaLive channels"
context: "MediaLive channels must be stopped to modify input attachments. Each input can be attached to one channel at a time. Channel class affects the number of supported inputs."
---

## Phase 1 — Triage

MUST:
- Check channel state and current attachments: `aws medialive describe-channel --channel-id <channel-id> --query '{State:State,ChannelClass:ChannelClass,InputAttachments:InputAttachments}'`
- Check if the input is already attached elsewhere: `aws medialive describe-input --input-id <input-id> --query '{State:State,AttachedChannels:AttachedChannels}'`
- Verify the channel is in IDLE state before attempting attachment changes

SHOULD:
- Verify the input type is INPUT_DEVICE
- Check the channel class supports the desired number of inputs

MAY:
- Review CloudTrail for recent UpdateChannel events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateChannel --max-results 10`

## Phase 2 — Remediate

MUST:
- Stop the channel before modifying attachments: `aws medialive stop-channel --channel-id <channel-id>`
- Wait for channel state to reach IDLE before making changes
- Detach input from existing channel before attaching to a new one

SHOULD:
- Use the console or update-channel API to modify input attachments
- Verify input attachment settings match the channel's encoding profile

MAY:
- Configure input switching schedules for multi-input channels
- Set up automatic input failover for STANDARD class channels

## Safety Ratings

```
safety_ratings:
  - "describe-channel: GREEN — read-only channel details"
  - "describe-input: GREEN — read-only input details"
  - "stop-channel: RED — interrupts live stream"
  - "update-channel input attachments: YELLOW — modifies channel configuration"
  - "start-channel: YELLOW — begins encoding and incurs costs"
```

## Escalation Conditions

- Channel fails to stop (stuck in STOPPING state)
- Input attachment update returns validation errors despite correct configuration
- Channel cannot start after input attachment changes
- Need to modify attachments on a channel that cannot be stopped (24/7 production)

## Data Sensitivity

- `aws medialive describe-channel`: MEDIUM — reveals channel configuration and output destinations
- `aws medialive describe-input`: LOW — returns input metadata
- `aws medialive stop-channel`: HIGH — interrupts live production stream
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER attempt to modify input attachments on a running channel
- NEVER force-stop a channel during live production without explicit approval
- NEVER delete an input while it is attached to a channel
- NEVER attach more inputs than the channel class supports

## Phase 3 — Rollback

1. If the channel was stopped for attachment changes, restart it: `aws medialive start-channel --channel-id <channel-id>`
2. If input attachments were modified incorrectly, stop the channel and restore previous configuration
3. If an input was detached from the wrong channel, reattach it after stopping that channel
4. Verify all affected channels are running correctly after rollback
5. Document the attachment change timeline and notify the production team

## Common Issues

- symptoms: "UpdateChannel returns ValidationException when adding input"
  diagnosis: "Channel is not in IDLE state — it must be stopped before modifying attachments."
  resolution: "Stop the channel first: aws medialive stop-channel --channel-id <id>. Wait for IDLE state."

## Output Format

```yaml
root_cause: "channel_attachment — <specific_cause>"
evidence:
  - type: attachment_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix channel attachment configuration"
  long_term: "Document attachment procedures and implement change management"
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
