---
title: "E1 — Formality Setting Issues"
description: "Diagnose formality setting problems"
status: active
severity: LOW
triggers:
  - "formality not working"
  - "formal translation"
  - "informal translation"
owner: devops-agent
objective: "Identify and resolve formality setting issues"
context: "Formality setting (FORMAL/INFORMAL) controls translation register. Only works for specific target languages: de, es, fr, hi, it, ja, nl, pl, pt. Silently ignored for other languages."
---

## Phase 1 — Triage

MUST:
- Verify target language supports formality (de, es, fr, hi, it, ja, nl, pl, pt)
- Test with formality: `aws translate translate-text --text "How are you?" --source-language-code en --target-language-code de --settings '{"Formality":"FORMAL"}'`
- Compare FORMAL vs INFORMAL output
- Check if formality is being silently ignored

SHOULD:
- Test with text that has clear formal/informal distinctions
- Verify Settings parameter format

MAY:
- Compare formality across supported languages

## Phase 2 — Remediate

MUST:
- Use formality only with supported target languages
- Set Formality in Settings parameter correctly

SHOULD:
- Document which languages support formality
- Default to FORMAL for business communications

MAY:
- Implement language-aware formality selection

## Common Issues

- symptoms: "Formality setting has no effect"
  diagnosis: "Target language doesn't support formality."
  resolution: "Check supported languages. Formality is silently ignored for unsupported languages."

- symptoms: "Translation uses wrong formality level"
  diagnosis: "Settings parameter format incorrect."
  resolution: "Use correct Settings format: {\"Formality\":\"FORMAL\"}"

## Output Format

```yaml
root_cause: "formality — <specific_cause>"
evidence:
  - type: translation_result
    content: "<translation with formality>"
severity: LOW
mitigation:
  immediate: "Use supported language or fix settings"
  long_term: "Implement language-aware formality handling"
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
