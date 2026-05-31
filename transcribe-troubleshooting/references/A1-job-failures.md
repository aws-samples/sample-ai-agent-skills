---
title: "A1 — Transcription Job Failures"
description: "Diagnose batch transcription job failures"
status: active
severity: HIGH
triggers:
  - "transcription job failed"
  - "StartTranscriptionJob error"
  - "job FAILED status"
owner: devops-agent
objective: "Identify and resolve transcription job failures"
context: "Batch transcription jobs process audio from S3. Common failures: wrong media format, S3 access denied, unsupported audio, or job name conflict."
---

## Phase 1 — Triage

MUST:
- Check job status: `aws transcribe get-transcription-job --transcription-job-name <name>`
- Review FailureReason in job details
- Verify audio file in S3: `aws s3 head-object --bucket <bucket> --key <key>`
- Check MediaFormat matches actual audio format

SHOULD:
- Verify S3 bucket is in same region
- Check audio duration (≤4 hours)
- Verify file size (≤2GB)

MAY:
- Check CloudTrail for StartTranscriptionJob errors
- Test with a known-good audio file

## Phase 2 — Remediate

MUST:
- Set correct MediaFormat matching actual audio
- Grant S3 read access for audio file
- Use unique job name

SHOULD:
- Validate audio file before submission
- Specify MediaSampleRateHertz for better accuracy

MAY:
- Implement job submission validation pipeline

## Common Issues

- symptoms: "Job fails with 'The media format that you specified doesn't match'"
  diagnosis: "MediaFormat parameter doesn't match actual audio format."
  resolution: "Check actual format. Set correct MediaFormat (wav, mp3, mp4, etc.)."

- symptoms: "ConflictException on StartTranscriptionJob"
  diagnosis: "Job name already exists."
  resolution: "Use unique job name. Include timestamp or UUID."

## Output Format

```yaml
root_cause: "job_failure — <specific_cause>"
evidence:
  - type: job_status
    content: "<job details and failure reason>"
severity: HIGH
mitigation:
  immediate: "Fix media format or S3 access"
  long_term: "Implement job submission validation"
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
