---
title: "F1 — Content Redaction Issues"
description: "Diagnose content redaction and PII removal problems"
status: active
severity: HIGH
triggers: ["redaction not working", "PII not redacted", "content redaction error"]
owner: devops-agent
objective: "Identify and resolve content redaction issues"
context: "Content redaction replaces PII in transcripts with [PII] tags. Available for specific languages (primarily English). Supports redaction in text and audio output."
---

## Phase 1 — Triage

MUST:
- Verify ContentRedaction settings in job configuration
- Check language supports redaction (en-US, en-AU, en-GB)
- Review redacted output for PII handling
- Verify RedactionType and RedactionOutput settings

SHOULD:
- Check PiiEntityTypes to redact
- Test with known PII content
- Verify both text and audio redaction if needed

MAY:
- Compare redacted vs unredacted output
- Check for missed PII

## Phase 2 — Remediate

MUST:
- Use supported language for content redaction
- Configure ContentRedaction in job settings
- Specify PiiEntityTypes to redact

SHOULD:
- Test redaction with representative content
- Implement human review for sensitive content

MAY:
- Build redaction validation pipeline

## Common Issues

- symptoms: "PII not redacted in transcript"
  diagnosis: "ContentRedaction not configured or language not supported."
  resolution: "Add ContentRedaction to job settings. Use supported language."

- symptoms: "Non-PII text redacted"
  diagnosis: "False positive PII detection."
  resolution: "Review PiiEntityTypes. Narrow types to reduce false positives."

## Output Format

```yaml
root_cause: "content_redaction — <specific_cause>"
evidence:
  - type: redaction_config
    content: "<content redaction settings>"
severity: HIGH
mitigation:
  immediate: "Fix redaction configuration"
  long_term: "Implement redaction validation pipeline"
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
