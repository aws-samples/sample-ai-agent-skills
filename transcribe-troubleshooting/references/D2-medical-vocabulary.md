---
title: "D2 — Medical Vocabulary Issues"
description: "Diagnose medical custom vocabulary problems"
status: active
severity: MEDIUM
triggers: ["medical vocabulary error", "medical terms wrong", "CreateMedicalVocabulary error"]
owner: devops-agent
objective: "Identify and resolve medical vocabulary issues"
context: "Medical custom vocabularies improve recognition of medical terms. Same format as standard vocabulary but used with medical transcription API."
---

## Phase 1 — Triage

MUST:
- Check vocabulary: `aws transcribe get-medical-vocabulary --vocabulary-name <name>`
- Verify vocabulary status is READY
- Check vocabulary format
- Verify vocabulary used in medical job settings

SHOULD:
- Include medical terminology with pronunciation hints
- Test with medical audio samples
- Compare with and without vocabulary

MAY:
- Review medical term recognition accuracy
- Check for vocabulary size limits

## Phase 2 — Remediate

MUST:
- Use correct vocabulary format with medical terms
- Wait for READY status
- Include vocabulary in medical transcription job

SHOULD:
- Add SoundsLike for complex medical terms
- Test iteratively with representative audio

MAY:
- Build medical vocabulary management pipeline

## Common Issues

- symptoms: "Medical vocabulary FAILED"
  diagnosis: "Format error in vocabulary file."
  resolution: "Fix table format. Check encoding."

- symptoms: "Medical terms still misrecognized"
  diagnosis: "Missing pronunciation hints for complex terms."
  resolution: "Add SoundsLike for medical terminology."

## Output Format

```yaml
root_cause: "medical_vocabulary — <specific_cause>"
evidence:
  - type: vocabulary_status
    content: "<medical vocabulary details>"
severity: MEDIUM
mitigation:
  immediate: "Fix vocabulary format or add pronunciation hints"
  long_term: "Maintain comprehensive medical vocabulary"
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
