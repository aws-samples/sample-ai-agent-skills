---
title: "C1 — Custom Vocabulary Creation Issues"
description: "Diagnose custom vocabulary creation and processing problems"
status: active
severity: MEDIUM
triggers: ["vocabulary creation failed", "vocabulary FAILED status", "CreateVocabulary error"]
owner: devops-agent
objective: "Identify and resolve vocabulary creation issues"
context: "Custom vocabularies improve recognition of specific terms. Support table format (phrases, IPA, SoundsLike, DisplayAs) or list format. Must reach READY state before use."
---

## Phase 1 — Triage

MUST:
- Check vocabulary status: `aws transcribe get-vocabulary --vocabulary-name <name>`
- Review FailureReason if status is FAILED
- Verify vocabulary file format
- Check language code matches intended use

SHOULD:
- Verify table format columns (Phrase, SoundsLike, IPA, DisplayAs)
- Check for encoding issues (UTF-8 required)
- Verify phrases are valid for the language

MAY:
- Compare with working vocabulary
- Check vocabulary quota limits

## Phase 2 — Remediate

MUST:
- Use correct table format with proper columns
- Use UTF-8 encoding
- Wait for READY status before using

SHOULD:
- Include SoundsLike or IPA for unusual pronunciations
- Test vocabulary with representative audio

MAY:
- Implement vocabulary validation pipeline

## Common Issues

- symptoms: "Vocabulary status FAILED"
  diagnosis: "Format error in vocabulary file."
  resolution: "Check table format. Verify column headers. Fix encoding."

- symptoms: "Vocabulary READY but terms not recognized"
  diagnosis: "Phrases don't match audio pronunciation."
  resolution: "Add SoundsLike hints. Test with representative audio."

## Output Format

```yaml
root_cause: "vocabulary_creation — <specific_cause>"
evidence:
  - type: vocabulary_status
    content: "<vocabulary details>"
severity: MEDIUM
mitigation:
  immediate: "Fix vocabulary format"
  long_term: "Implement vocabulary testing pipeline"
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
