---
title: "F1 — Document Quality Issues"
description: "Diagnose document quality impact on extraction accuracy"
status: active
severity: MEDIUM
triggers:
  - "low accuracy"
  - "poor extraction quality"
  - "blurry document"
  - "skewed document"
owner: devops-agent
objective: "Identify and resolve document quality issues"
context: "Document quality directly affects extraction accuracy. Low resolution, skew, poor lighting, and handwriting reduce confidence scores. Preprocessing can improve results."
---

## Phase 1 — Triage

MUST:
- Check overall confidence scores in extraction results
- Verify document resolution (300 DPI recommended)
- Check for document skew or rotation
- Verify document is not password-protected

SHOULD:
- Compare confidence across different document regions
- Check for handwritten vs printed content
- Test with enhanced version of same document

MAY:
- Assess document quality programmatically
- Compare results before and after preprocessing

## Phase 2 — Remediate

MUST:
- Use minimum 300 DPI resolution for scanned documents
- Ensure documents are properly oriented
- Use clear, high-contrast documents

SHOULD:
- Implement document preprocessing (deskew, contrast enhancement)
- Set confidence thresholds for automated processing

MAY:
- Build document quality assessment pipeline

## Common Issues

- symptoms: "Confidence scores below 80% across document"
  diagnosis: "Low resolution scan or poor document quality."
  resolution: "Rescan at higher resolution. Apply image enhancement."

- symptoms: "Text detected but garbled"
  diagnosis: "Document heavily skewed or rotated."
  resolution: "Deskew document before processing."

## Output Format

```yaml
root_cause: "document_quality — <specific_cause>"
evidence:
  - type: confidence_scores
    content: "<average confidence by region>"
  - type: quality_assessment
    content: "<document quality factors>"
severity: MEDIUM
mitigation:
  immediate: "Improve document quality"
  long_term: "Implement document preprocessing pipeline"
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
