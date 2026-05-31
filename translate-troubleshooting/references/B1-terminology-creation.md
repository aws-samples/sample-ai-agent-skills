---
title: "B1 — Custom Terminology Creation Issues"
description: "Diagnose custom terminology creation and import problems"
status: active
severity: MEDIUM
triggers:
  - "terminology creation failed"
  - "ImportTerminology error"
  - "terminology format error"
owner: devops-agent
objective: "Identify and resolve terminology creation issues"
context: "Custom terminologies define specific term translations. Support CSV and TMX formats. Terms are case-sensitive. Maximum 256 terms per resource (default)."
---

## Phase 1 — Triage

MUST:
- List terminologies: `aws translate list-terminologies`
- Check terminology: `aws translate get-terminology --name <name> --terminology-data-format CSV`
- Verify terminology file format (CSV or TMX)
- Check term count against limits

SHOULD:
- Verify CSV format: source language column, target language columns
- Check for encoding issues in terminology file
- Verify language codes in terminology match translation request

MAY:
- Review CloudTrail for ImportTerminology events
- Check for duplicate terms

## Phase 2 — Remediate

MUST:
- Use correct CSV format (en,es,fr header with language codes)
- Ensure UTF-8 encoding
- Stay within term count limits

SHOULD:
- Include all case variations for case-sensitive matching
- Test terminology with sample translations

MAY:
- Implement terminology management pipeline

## Common Issues

- symptoms: "ImportTerminology fails with format error"
  diagnosis: "CSV format incorrect or encoding issues."
  resolution: "Use UTF-8 CSV with language code headers. Check column alignment."

- symptoms: "Terminology imported but term count is 0"
  diagnosis: "All terms failed validation."
  resolution: "Check term format. Ensure source and target columns have values."

## Output Format

```yaml
root_cause: "terminology_creation — <specific_cause>"
evidence:
  - type: terminology_info
    content: "<terminology details>"
severity: MEDIUM
mitigation:
  immediate: "Fix terminology file format"
  long_term: "Implement terminology validation pipeline"
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
