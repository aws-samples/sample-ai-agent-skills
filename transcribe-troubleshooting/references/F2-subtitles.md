---
title: "F2 — Subtitle Generation Issues"
description: "Diagnose subtitle (SRT/VTT) generation problems"
status: active
severity: LOW
triggers: ["subtitles not generated", "SRT format error", "VTT output missing"]
owner: devops-agent
objective: "Identify and resolve subtitle generation issues"
context: "Transcribe can generate SRT and VTT subtitle files. Requires Subtitles configuration in job settings. Output includes timestamps for each caption segment."
---

## Phase 1 — Triage

MUST:
- Verify Subtitles configuration in job settings
- Check subtitle format requested (SRT, VTT)
- Verify job completed successfully
- Check output location for subtitle files

SHOULD:
- Review subtitle timing accuracy
- Check for encoding issues in subtitle output
- Verify subtitle segment length

MAY:
- Compare subtitle output with transcript
- Test with different subtitle formats

## Phase 2 — Remediate

MUST:
- Add Subtitles configuration to job settings
- Specify desired format (SRT and/or VTT)
- Check output S3 location for subtitle files

SHOULD:
- Validate subtitle timing with video
- Check for character encoding issues

MAY:
- Implement subtitle post-processing pipeline

## Common Issues

- symptoms: "No subtitle file in output"
  diagnosis: "Subtitles not configured in job settings."
  resolution: "Add Subtitles:{Formats:[\"srt\",\"vtt\"]} to job settings."

- symptoms: "Subtitle timing off"
  diagnosis: "Audio processing timing differences."
  resolution: "Review and adjust subtitle timing in post-processing."

## Output Format

```yaml
root_cause: "subtitles — <specific_cause>"
evidence:
  - type: subtitle_config
    content: "<subtitle settings>"
severity: LOW
mitigation:
  immediate: "Fix subtitle configuration"
  long_term: "Implement subtitle quality validation"
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
