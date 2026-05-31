---
title: "E1 — Call Analytics Job Issues"
description: "Diagnose Call Analytics job failures"
status: active
severity: HIGH
triggers: ["call analytics failed", "StartCallAnalyticsJob error", "call analytics error"]
owner: devops-agent
objective: "Identify and resolve Call Analytics job failures"
context: "Call Analytics transcribes and analyzes call recordings. Requires stereo audio with separate channels per speaker. Provides sentiment, issues, and call summarization."
---

## Phase 1 — Triage

MUST:
- Check job: `aws transcribe get-call-analytics-job --call-analytics-job-name <name>`
- Verify audio is stereo (2 channels)
- Check ChannelDefinitions configuration
- Review failure reason

SHOULD:
- Verify audio format and sample rate
- Check categories configuration if used
- Verify IAM permissions

MAY:
- Test with known stereo call recording
- Check for concurrent job limits

## Phase 2 — Remediate

MUST:
- Use stereo audio with separate channels per speaker
- Configure ChannelDefinitions (channel 0 = agent, channel 1 = customer, or vice versa)
- Set correct MediaFormat

SHOULD:
- Create call analytics categories for issue detection
- Configure output location

MAY:
- Implement call analytics pipeline

## Common Issues

- symptoms: "Call analytics fails with channel error"
  diagnosis: "Mono audio provided instead of stereo."
  resolution: "Use stereo recording with separate channels per speaker."

- symptoms: "Speaker labels swapped"
  diagnosis: "ChannelDefinitions maps channels incorrectly."
  resolution: "Swap channel assignments in ChannelDefinitions."

## Output Format

```yaml
root_cause: "call_analytics — <specific_cause>"
evidence:
  - type: job_status
    content: "<call analytics job details>"
severity: HIGH
mitigation:
  immediate: "Fix audio format or channel configuration"
  long_term: "Implement call recording standards for analytics"
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
