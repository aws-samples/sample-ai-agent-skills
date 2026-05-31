---
title: "A2 — Language Pair Issues"
description: "Diagnose language pair support and detection problems"
status: active
severity: MEDIUM
triggers:
  - "language not supported"
  - "language detection wrong"
  - "auto detect failed"
owner: devops-agent
objective: "Identify and resolve language pair issues"
context: "Amazon Translate supports 75+ languages but not all pairs. Auto-detection adds latency and may be inaccurate for short text. Some pairs use English as pivot."
---

## Phase 1 — Triage

MUST:
- List supported languages: `aws translate list-languages`
- Verify source and target language codes
- Test with explicit language codes (not auto)
- Check if language pair is directly supported

SHOULD:
- Test auto-detection with representative text
- Check for language code format (BCP-47)
- Verify regional language variants (pt vs pt-BR)

MAY:
- Test translation quality for the specific pair
- Check for pivot language behavior

## Phase 2 — Remediate

MUST:
- Use supported language codes
- Specify source language when known (avoid auto for short text)
- Use correct language code format

SHOULD:
- Implement language detection as preprocessing
- Handle unsupported pairs with English pivot

MAY:
- Build language pair validation into application

## Common Issues

- symptoms: "UnsupportedLanguagePairException"
  diagnosis: "Direct pair not supported."
  resolution: "Translate source→English, then English→target."

- symptoms: "Auto-detection returns wrong language"
  diagnosis: "Text too short or ambiguous."
  resolution: "Use longer text or specify source language explicitly."

## Output Format

```yaml
root_cause: "language_pairs — <specific_cause>"
evidence:
  - type: supported_languages
    content: "<language support check>"
severity: MEDIUM
mitigation:
  immediate: "Use correct language codes"
  long_term: "Implement language validation pipeline"
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

- Source documents may contain business-sensitive content
- Custom terminologies reveal domain-specific vocabulary
- Parallel data contains paired translation examples
- Classification: CONFIDENTIAL - business content and proprietary terminology

## Prohibited Actions

- NEVER delete custom terminologies used by production translation workflows
- NEVER modify parallel data sets during active batch translation jobs
- NEVER change formality settings on production translation endpoints without review
- NEVER process documents containing PII without confirming data handling compliance

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
