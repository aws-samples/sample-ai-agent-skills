---
title: "A2 — Audio Format Issues"
description: "Diagnose audio format and quality problems"
status: active
severity: MEDIUM
triggers:
  - "audio format error"
  - "unsupported format"
  - "poor transcription quality"
  - "sample rate issue"
owner: devops-agent
objective: "Identify and resolve audio format issues"
context: "Transcribe supports WAV, MP3, MP4, FLAC, OGG, AMR, WebM. Sample rate affects accuracy. 16kHz+ recommended. Audio quality directly impacts transcription quality."
---

## Phase 1 — Triage

MUST:
- Verify audio format matches MediaFormat parameter
- Check audio sample rate
- Verify audio is not corrupted
- Test with a known-good audio file

SHOULD:
- Check audio channels (mono vs stereo)
- Verify audio encoding (PCM, MP3, etc.)
- Test with different sample rates

MAY:
- Use ffprobe or similar to inspect audio properties
- Compare transcription quality across formats

## Phase 2 — Remediate

MUST:
- Convert to supported format if needed
- Use 16kHz+ sample rate for best accuracy
- Specify correct MediaFormat

SHOULD:
- Use WAV or FLAC for highest quality
- Specify MediaSampleRateHertz explicitly

MAY:
- Implement audio preprocessing pipeline

## Common Issues

- symptoms: "Transcription has many errors"
  diagnosis: "Low sample rate or poor audio quality."
  resolution: "Use 16kHz+ sample rate. Improve audio quality."

- symptoms: "Job fails with format error"
  diagnosis: "File extension doesn't match actual encoding."
  resolution: "Check actual encoding. Convert if needed."

## Output Format

```yaml
root_cause: "audio_format — <specific_cause>"
evidence:
  - type: audio_properties
    content: "<audio format and sample rate>"
severity: MEDIUM
mitigation:
  immediate: "Fix audio format or sample rate"
  long_term: "Implement audio quality validation"
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
