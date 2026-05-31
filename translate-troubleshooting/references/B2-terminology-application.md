---
title: "B2 — Terminology Application Issues"
description: "Diagnose custom terminology not being applied in translations"
status: active
severity: MEDIUM
triggers:
  - "terminology not applied"
  - "custom terms ignored"
  - "term not translated correctly"
owner: devops-agent
objective: "Identify and resolve terminology application issues"
context: "Custom terminology overrides neural translation for matched terms. Matching is case-sensitive. Terminology must be specified in the translation request."
---

## Phase 1 — Triage

MUST:
- Verify terminology is specified in request: `aws translate translate-text --text "<text>" --source-language-code en --target-language-code es --terminology-names <name>`
- Check term exists in terminology for the language pair
- Verify case matches exactly (case-sensitive)
- Check AppliedTerminologies in response

SHOULD:
- Test with exact term text from terminology
- Verify terminology language codes match request
- Check for partial term matching behavior

MAY:
- Test with different text contexts
- Compare with and without terminology

## Phase 2 — Remediate

MUST:
- Include terminology name in TerminologyNames parameter
- Match source text case exactly with terminology entry
- Verify terminology has entry for the language pair

SHOULD:
- Add case variations to terminology
- Test terminology application before production use

MAY:
- Implement terminology testing automation

## Common Issues

- symptoms: "Term not applied despite being in terminology"
  diagnosis: "Case mismatch between source text and terminology entry."
  resolution: "Add all case variations. Match exact casing."

- symptoms: "Terminology applied but translation quality degraded"
  diagnosis: "Forced term translation disrupts sentence structure."
  resolution: "Review term context. Adjust terminology entries for natural flow."

## Output Format

```yaml
root_cause: "terminology_application — <specific_cause>"
evidence:
  - type: applied_terminologies
    content: "<AppliedTerminologies from response>"
severity: MEDIUM
mitigation:
  immediate: "Fix case matching or terminology configuration"
  long_term: "Implement terminology testing and quality assurance"
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
