---
title: "C2 — Multi-Page Processing Issues"
description: "Diagnose multi-page document processing problems"
status: active
severity: MEDIUM
triggers:
  - "multi-page not working"
  - "pages missing"
  - "pagination error"
  - "NextToken issue"
owner: devops-agent
objective: "Identify and resolve multi-page processing issues"
context: "Multi-page documents require async APIs. Results are paginated with NextToken. Each page's blocks include page number. Maximum 3000 pages per document."
---

## Phase 1 — Triage

MUST:
- Verify using async API (not sync): `aws textract start-document-analysis --document-location '{"S3Object":{"Bucket":"<bucket>","Name":"<key>"}}' --feature-types '["FORMS","TABLES"]'`
- Check all pages returned: follow NextToken pagination
- Verify document page count
- Check for page-specific errors

SHOULD:
- Verify document format supports multi-page (PDF, TIFF)
- Check if specific pages have issues
- Monitor processing time for large documents

MAY:
- Split very large documents for parallel processing
- Check page-level confidence scores

## Phase 2 — Remediate

MUST:
- Use async APIs for multi-page documents
- Follow NextToken pagination to get all results
- Handle per-page block organization

SHOULD:
- Implement pagination handling in result processing
- Track page numbers in extracted data

MAY:
- Implement parallel processing for very large documents

## Common Issues

- symptoms: "Only partial pages returned"
  diagnosis: "Not following NextToken pagination."
  resolution: "Loop through GetDocumentAnalysis with NextToken until no more tokens."

- symptoms: "Sync API returns only first page"
  diagnosis: "Using sync API for multi-page document."
  resolution: "Switch to StartDocumentAnalysis (async) for multi-page."

## Output Format

```yaml
root_cause: "multipage — <specific_cause>"
evidence:
  - type: page_count
    content: "<document page count>"
  - type: pagination
    content: "<NextToken handling>"
severity: MEDIUM
mitigation:
  immediate: "Use async API with proper pagination"
  long_term: "Implement robust multi-page processing pipeline"
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
