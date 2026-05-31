---
title: "C2 — Custom Translation Model Issues"
description: "Diagnose custom model training and quality problems"
status: active
severity: MEDIUM
triggers:
  - "custom model not improving"
  - "parallel data training failed"
  - "custom translation quality"
owner: devops-agent
objective: "Identify and resolve custom model issues"
context: "Custom models are trained using parallel data in batch translation jobs. Training happens automatically. Quality improvement depends on data quality and relevance."
---

## Phase 1 — Triage

MUST:
- Check parallel data status: `aws translate get-parallel-data --name <name>`
- Verify batch job used parallel data: `aws translate describe-text-translation-job --job-id <job-id>`
- Compare translation quality with and without parallel data
- Check parallel data record count

SHOULD:
- Verify parallel data is domain-relevant
- Check language pair matches
- Review translation samples

MAY:
- Test with different parallel data sets
- Compare BLEU scores if available

## Phase 2 — Remediate

MUST:
- Use domain-relevant parallel data
- Provide sufficient examples (100+ sentences)
- Verify language pair alignment

SHOULD:
- Use high-quality human translations as parallel data
- Test custom model on representative samples

MAY:
- Implement translation quality evaluation pipeline

## Common Issues

- symptoms: "Custom model doesn't improve translation"
  diagnosis: "Parallel data not domain-relevant or insufficient."
  resolution: "Use domain-specific parallel data. Add more examples."

- symptoms: "Custom model degrades general translation"
  diagnosis: "Parallel data too narrow, causing overfitting."
  resolution: "Include diverse examples. Balance domain-specific and general text."

## Output Format

```yaml
root_cause: "custom_models — <specific_cause>"
evidence:
  - type: parallel_data
    content: "<parallel data details>"
  - type: translation_quality
    content: "<quality comparison>"
severity: MEDIUM
mitigation:
  immediate: "Improve parallel data quality"
  long_term: "Implement translation quality evaluation"
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
