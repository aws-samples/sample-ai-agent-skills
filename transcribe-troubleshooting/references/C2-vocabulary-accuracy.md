---
title: "C2 — Vocabulary Accuracy Issues"
description: "Diagnose custom vocabulary effectiveness problems"
status: active
severity: MEDIUM
triggers: ["vocabulary not improving", "terms still wrong", "custom words not recognized"]
owner: devops-agent
objective: "Improve custom vocabulary effectiveness"
context: "Custom vocabulary effectiveness depends on phrase format, SoundsLike hints, and audio quality. Vocabulary boosts recognition probability but doesn't guarantee 100% accuracy."
---

## Phase 1 — Triage

MUST:
- Verify vocabulary is applied to job: check VocabularyName in job config
- Compare transcription with and without vocabulary
- Check vocabulary phrases match expected terms
- Verify vocabulary language matches audio language

SHOULD:
- Add SoundsLike hints for unusual pronunciations
- Check DisplayAs for correct output formatting
- Test with clear audio of target terms

MAY:
- Use vocabulary filter to suppress unwanted words
- Compare across different audio samples

## Phase 2 — Remediate

MUST:
- Include vocabulary name in transcription job settings
- Add SoundsLike for terms with unusual pronunciation
- Match vocabulary language to audio language

SHOULD:
- Use DisplayAs for proper formatting (acronyms, proper nouns)
- Test vocabulary iteratively with representative audio

MAY:
- Implement vocabulary effectiveness tracking

## Common Issues

- symptoms: "Custom term still transcribed incorrectly"
  diagnosis: "Missing SoundsLike hint or audio quality too low."
  resolution: "Add SoundsLike pronunciation. Improve audio quality."

- symptoms: "Vocabulary applied but no improvement"
  diagnosis: "Phrases don't match how terms are spoken."
  resolution: "Adjust phrases to match spoken form. Add pronunciation hints."

## Output Format

```yaml
root_cause: "vocabulary_accuracy — <specific_cause>"
evidence:
  - type: transcription_comparison
    content: "<with vs without vocabulary>"
severity: MEDIUM
mitigation:
  immediate: "Add pronunciation hints"
  long_term: "Implement vocabulary tuning process"
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
