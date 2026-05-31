---
title: "C1 — Channel Assembly Issues"
description: "Diagnose channel assembly channel creation and playback issues"
status: active
severity: HIGH
triggers: ["channel not playing", "channel assembly error", "program schedule failed", "channel creation failed"]
owner: devops-agent
objective: "Resolve channel assembly issues for linear channel delivery"
context: "Channel assembly creates linear channels from VOD sources. Channels have programs scheduled in order, each referencing VOD sources from source locations."
---

## Phase 1 — Triage

MUST:
- List channels and check state: `aws mediatailor list-channels --query 'Items[*].{Name:ChannelName,State:ChannelState,Tier:Tier}'`
- Describe the channel: `aws mediatailor describe-channel --ChannelName <channel-name>`
- Check channel policy: `aws mediatailor get-channel-policy --ChannelName <channel-name>`

SHOULD:
- List programs in the channel: `aws mediatailor get-channel-schedule --ChannelName <channel-name>`
- Verify source locations referenced by programs are accessible
- Check channel output configuration (HLS/DASH manifest settings)

MAY:
- Review CloudWatch metrics for channel playback errors
- Test channel playback URL directly

## Phase 2 — Remediate

MUST:
- Start channel if stopped: `aws mediatailor start-channel --ChannelName <channel-name>`
- Fix source location references in programs if sources are inaccessible
- Ensure channel has at least one program in the schedule

SHOULD:
- Verify VOD source durations match program schedule expectations
- Configure channel outputs for required manifest formats (HLS/DASH)

MAY:
- Adjust channel tier (BASIC vs STANDARD) based on requirements
- Configure channel filler slate for schedule gaps

## Safety Ratings

```
safety_ratings:
  - "list-channels: GREEN — read-only enumeration"
  - "describe-channel: GREEN — read-only channel details"
  - "start-channel: YELLOW — starts channel playback, consumes resources"
  - "stop-channel: RED — immediately stops channel, breaks active viewer sessions"
  - "delete-channel: RED — permanently removes channel and schedule"
```

## Escalation Conditions

- Channel fails to start with internal service errors after fixing all configuration
- Multiple channels experience simultaneous playback failures
- Source locations become inaccessible affecting multiple channels
- Channel schedule requires changes during active live linear broadcast

## Data Sensitivity

- `aws mediatailor list-channels`: LOW — returns channel metadata only
- `aws mediatailor describe-channel`: LOW — channel configuration details
- `aws mediatailor get-channel-schedule`: LOW — program schedule information
- Channel playback URLs: MEDIUM — contain account-specific endpoint information

## Prohibited Actions

- NEVER stop a channel during active viewer sessions without stakeholder approval
- NEVER delete a channel without first stopping it and confirming no active viewers
- NEVER modify a channel schedule during a live broadcast window
- NEVER remove source locations referenced by active channel programs

## Phase 3 — Rollback

1. If channel was stopped accidentally, restart: `aws mediatailor start-channel --ChannelName <channel-name>`
2. If program schedule was modified incorrectly, update programs with correct source references
3. If channel output configuration was changed, update outputs and restart channel
4. Monitor channel playback URL for 5 minutes after rollback to confirm recovery

## Common Issues

- symptoms: "Channel state is STOPPED and will not start"
  diagnosis: "Channel has no programs or references invalid source locations."
  resolution: "Add at least one program with valid VOD source. Verify source locations exist and are accessible."

- symptoms: "Channel plays but has gaps between programs"
  diagnosis: "Program schedule has gaps or VOD source durations are shorter than scheduled."
  resolution: "Configure filler slate. Adjust program durations to match VOD source lengths."

## Output Format

```yaml
root_cause: "channel_assembly — <specific_cause>"
evidence:
  - type: channel_detail
    content: "<channel configuration and state>"
severity: HIGH
mitigation:
  immediate: "Fix channel configuration"
  long_term: "Implement channel health monitoring"
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
