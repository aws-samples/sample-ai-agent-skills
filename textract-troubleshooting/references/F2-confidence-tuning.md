---
title: "F2 — Confidence Score Tuning"
description: "Diagnose and optimize confidence threshold settings"
status: active
severity: LOW
triggers:
  - "confidence threshold"
  - "false positives"
  - "false negatives"
  - "accuracy tuning"
owner: devops-agent
objective: "Optimize confidence thresholds for extraction accuracy"
context: "Textract returns confidence scores (0-100) for each extracted element. Threshold tuning balances false positives vs false negatives. Different use cases need different thresholds."
---

## Phase 1 — Triage

MUST:
- Review confidence score distribution in results
- Identify fields with consistently low confidence
- Check if threshold is appropriate for use case
- Compare automated results with manual verification

SHOULD:
- Analyze confidence patterns across document types
- Check for systematic low-confidence areas
- Test different threshold values

MAY:
- Build confidence score analytics
- Compare across document batches

## Phase 2 — Remediate

MUST:
- Set threshold based on use case (higher for automated, lower for human review)
- Implement human review for below-threshold results
- Document threshold decisions

SHOULD:
- Use different thresholds for different field types
- Implement confidence-based routing (auto-accept, review, reject)

MAY:
- Build adaptive threshold system

## Common Issues

- symptoms: "Too many results flagged for review"
  diagnosis: "Confidence threshold too high."
  resolution: "Lower threshold. Improve document quality to increase confidence."

- symptoms: "Incorrect values accepted automatically"
  diagnosis: "Confidence threshold too low."
  resolution: "Raise threshold. Add validation rules beyond confidence."

## Output Format

```yaml
root_cause: "confidence_tuning — <specific_cause>"
evidence:
  - type: confidence_distribution
    content: "<confidence score distribution>"
  - type: threshold_analysis
    content: "<threshold impact analysis>"
severity: LOW
mitigation:
  immediate: "Adjust confidence thresholds"
  long_term: "Implement adaptive confidence management"
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

- Documents may contain PII, financial data, or medical records
- Extracted text preserves all sensitive content from source documents
- Identity document analysis processes government-issued IDs
- Classification: HIGHLY CONFIDENTIAL - document content may contain regulated PII

## Prohibited Actions

- NEVER process documents containing PII without confirming data handling compliance
- NEVER delete async job results before downstream consumers have retrieved them
- NEVER modify confidence thresholds in production without validation testing
- NEVER store extracted document data without appropriate encryption

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
