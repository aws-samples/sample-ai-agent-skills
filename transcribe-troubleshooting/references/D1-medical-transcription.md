---
title: "D1 — Medical Transcription Issues"
description: "Diagnose medical transcription job failures"
status: active
severity: HIGH
triggers: ["medical transcription failed", "StartMedicalTranscriptionJob error", "medical job FAILED"]
owner: devops-agent
objective: "Identify and resolve medical transcription issues"
context: "Medical transcription uses separate HIPAA-compliant API. Supports specific medical specialties. Requires S3 output with encryption. Different from standard transcription."
---

## Phase 1 — Triage

MUST:
- Check job: `aws transcribe get-medical-transcription-job --medical-transcription-job-name <name>`
- Verify using medical API (not standard)
- Check Specialty parameter (PRIMARYCARE)
- Verify Type parameter (CONVERSATION or DICTATION)

SHOULD:
- Check audio format and quality
- Verify S3 output bucket has encryption
- Check IAM permissions for medical transcription

MAY:
- Compare with standard transcription results
- Check medical vocabulary if used

## Phase 2 — Remediate

MUST:
- Use StartMedicalTranscriptionJob (not standard API)
- Set correct Specialty and Type
- Configure encrypted S3 output

SHOULD:
- Use medical custom vocabulary for domain terms
- Specify correct MediaFormat

MAY:
- Implement medical transcription pipeline with validation

## Common Issues

- symptoms: "Medical job fails with parameter error"
  diagnosis: "Missing required Specialty or Type parameter."
  resolution: "Set Specialty=PRIMARYCARE and Type=CONVERSATION or DICTATION."

- symptoms: "Medical terms not recognized"
  diagnosis: "Standard API used instead of medical, or no medical vocabulary."
  resolution: "Use medical API. Add medical custom vocabulary."

## Output Format

```yaml
root_cause: "medical_transcription — <specific_cause>"
evidence:
  - type: job_status
    content: "<medical job details>"
severity: HIGH
mitigation:
  immediate: "Fix API or parameters"
  long_term: "Implement medical transcription pipeline"
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
