---
title: "E2 — Channel Configuration Issues"
description: "Diagnose audio channel and speaker identification problems"
status: active
severity: MEDIUM
triggers: ["channel identification error", "speaker wrong channel", "stereo config issue"]
owner: devops-agent
objective: "Identify and resolve channel configuration issues"
context: "Channel identification maps audio channels to speakers. Used in Call Analytics and standard transcription. Cannot use simultaneously with speaker diarization."
---

## Phase 1 — Triage

MUST:
- Verify audio is stereo with separate channels
- Check ChannelIdentification setting in job
- Review channel-to-speaker mapping
- Verify audio channels contain expected speakers

SHOULD:
- Test audio channels independently
- Check for cross-talk between channels
- Verify channel numbering (0-indexed)

MAY:
- Compare with speaker diarization results
- Check audio recording setup

## Phase 2 — Remediate

MUST:
- Use stereo audio with clean channel separation
- Enable ChannelIdentification in job settings
- Map channels correctly to speakers

SHOULD:
- Ensure minimal cross-talk between channels
- Test with sample recordings

MAY:
- Implement audio quality validation for channel separation

## Common Issues

- symptoms: "Both speakers on same channel"
  diagnosis: "Audio recorded in mono or channels mixed."
  resolution: "Use stereo recording with separate channels per speaker."

- symptoms: "Channel identification returns empty for one channel"
  diagnosis: "One channel is silent or has only noise."
  resolution: "Verify both channels have speech. Check recording setup."

## Output Format

```yaml
root_cause: "channel_config — <specific_cause>"
evidence:
  - type: channel_analysis
    content: "<channel configuration and results>"
severity: MEDIUM
mitigation:
  immediate: "Fix audio channels or configuration"
  long_term: "Standardize recording setup for channel separation"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Audio recordings may contain private conversations
- Transcription output preserves all spoken content including PII
- Medical transcriptions contain protected health information (PHI)
- Classification: HIGHLY CONFIDENTIAL - audio content and potential PHI/PII

## Prohibited Actions

- NEVER process audio containing PII without content redaction enabled
- NEVER delete custom vocabularies used by production transcription jobs
- NEVER modify medical transcription settings without clinical team approval
- NEVER store transcription output containing sensitive data without encryption

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
