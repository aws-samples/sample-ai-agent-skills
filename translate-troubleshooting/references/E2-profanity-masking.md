---
title: "E2 — Profanity Masking Issues"
description: "Diagnose profanity masking problems"
status: active
severity: LOW
triggers:
  - "profanity not masked"
  - "profanity masking error"
  - "grawlix in output"
owner: devops-agent
objective: "Identify and resolve profanity masking issues"
context: "Profanity masking replaces profane words with grawlix (?$#@$). Pattern-based, not contextual. Enable via Settings parameter. Detection varies by language."
---

## Phase 1 — Triage

MUST:
- Test with profanity masking: `aws translate translate-text --text "<text>" --source-language-code en --target-language-code es --settings '{"Profanity":"MASK"}'`
- Verify Settings parameter includes Profanity:MASK
- Check if profanity is detected in source or target language
- Review masked output

SHOULD:
- Test with known profane words
- Check language-specific profanity detection
- Verify masking doesn't affect non-profane words

MAY:
- Compare masked vs unmasked output
- Test across different language pairs

## Phase 2 — Remediate

MUST:
- Set Profanity to MASK in Settings parameter
- Understand masking is pattern-based (not contextual)

SHOULD:
- Implement additional content filtering for edge cases
- Test masking with representative content

MAY:
- Build content moderation pipeline beyond profanity masking

## Common Issues

- symptoms: "Profanity not masked in translation"
  diagnosis: "Settings parameter missing or profanity not in detection dictionary."
  resolution: "Add Profanity:MASK to Settings. Note: not all profanity detected."

- symptoms: "Non-profane words masked"
  diagnosis: "Pattern matching false positive."
  resolution: "Review masked words. Implement post-processing to restore false positives."

## Output Format

```yaml
root_cause: "profanity_masking — <specific_cause>"
evidence:
  - type: masked_output
    content: "<translation with masking>"
severity: LOW
mitigation:
  immediate: "Fix settings or handle false positives"
  long_term: "Implement comprehensive content moderation"
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
