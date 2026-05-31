---
title: "A1 — Translation Errors"
description: "Diagnose real-time translation failures"
status: active
severity: HIGH
triggers:
  - "translation failed"
  - "TranslateText error"
  - "translation quality poor"
owner: devops-agent
objective: "Identify and resolve translation errors"
context: "TranslateText provides real-time neural machine translation. Errors arise from text size limits, unsupported language pairs, encoding issues, or service throttling."
---

## Phase 1 — Triage

MUST:
- Test translation: `aws translate translate-text --text "Hello world" --source-language-code en --target-language-code es`
- Check text size (≤10,000 bytes UTF-8)
- Verify language pair is supported: `aws translate list-languages`
- Check error message details

SHOULD:
- Verify text encoding is UTF-8
- Check for special characters causing issues
- Test with simpler text to isolate problem

MAY:
- Check API throttling metrics
- Review CloudTrail for TranslateText errors

## Phase 2 — Remediate

MUST:
- Use supported language pair
- Keep text within 10KB limit
- Use UTF-8 encoding

SHOULD:
- Split large text into chunks for real-time API
- Use batch translation for large volumes

MAY:
- Implement translation error handling and retry

## Common Issues

- symptoms: "TextSizeLimitExceededException"
  diagnosis: "Text exceeds 10,000 byte limit."
  resolution: "Split text into smaller chunks or use batch translation."

- symptoms: "UnsupportedLanguagePairException"
  diagnosis: "Language pair not supported."
  resolution: "Check supported pairs. Use English as pivot language."

## Output Format

```yaml
root_cause: "translation_error — <specific_cause>"
evidence:
  - type: api_response
    content: "<TranslateText error>"
severity: HIGH
mitigation:
  immediate: "Fix text size or language pair"
  long_term: "Implement translation pipeline with validation"
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
