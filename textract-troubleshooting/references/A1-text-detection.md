---
title: "A1 — Text Detection Issues"
description: "Diagnose document text detection problems"
status: active
severity: MEDIUM
triggers:
  - "text not detected"
  - "DetectDocumentText error"
  - "missing text in output"
owner: devops-agent
objective: "Identify and resolve text detection issues"
context: "DetectDocumentText extracts raw text from documents. Sync API is single-page only. Returns LINE and WORD blocks with confidence scores."
---

## Phase 1 — Triage

MUST:
- Test text detection: `aws textract detect-document-text --document '{"S3Object":{"Bucket":"<bucket>","Name":"<key>"}}'`
- Verify document format (JPEG, PNG, PDF, TIFF)
- Check document size limits
- Review returned text blocks and confidence scores

SHOULD:
- Check document quality (resolution, clarity)
- Verify single-page for sync API
- Test with different document regions

MAY:
- Compare sync vs async results for same document
- Check for document orientation issues

## Phase 2 — Remediate

MUST:
- Use supported document format
- Use async API for multi-page documents
- Ensure document is readable quality

SHOULD:
- Preprocess documents (deskew, enhance contrast)
- Filter results by confidence threshold

MAY:
- Implement document quality assessment pipeline

## Common Issues

- symptoms: "Only first page text returned from PDF"
  diagnosis: "Using sync API which processes first page only."
  resolution: "Use StartDocumentTextDetection for multi-page PDFs."

- symptoms: "Text detection returns low confidence"
  diagnosis: "Poor document quality (low resolution, skewed, blurry)."
  resolution: "Improve document quality. Preprocess before analysis."

## Output Format

```yaml
root_cause: "text_detection — <specific_cause>"
evidence:
  - type: detection_result
    content: "<text detection response>"
  - type: document_info
    content: "<document format and quality>"
severity: MEDIUM
mitigation:
  immediate: "Fix document format or use async API"
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
