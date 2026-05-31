---
title: "A2 — Document Analysis Issues"
description: "Diagnose AnalyzeDocument API problems"
status: active
severity: HIGH
triggers:
  - "AnalyzeDocument error"
  - "analysis failed"
  - "feature type error"
owner: devops-agent
objective: "Identify and resolve document analysis issues"
context: "AnalyzeDocument extracts forms, tables, queries, and signatures. Requires explicit FeatureTypes. Sync is single-page; async for multi-page."
---

## Phase 1 — Triage

MUST:
- Test analysis: `aws textract analyze-document --document '{"S3Object":{"Bucket":"<bucket>","Name":"<key>"}}' --feature-types '["FORMS","TABLES"]'`
- Verify FeatureTypes parameter is correct
- Check document format and size
- Review block types in response

SHOULD:
- Test each feature type separately to isolate issues
- Verify document has the expected structure (forms, tables)
- Check for UnsupportedDocumentException

MAY:
- Compare results with DetectDocumentText
- Check API throttling

## Phase 2 — Remediate

MUST:
- Specify correct FeatureTypes (FORMS, TABLES, QUERIES, SIGNATURES)
- Use supported document format
- Use async for multi-page documents

SHOULD:
- Combine feature types as needed
- Handle block relationships in response parsing

MAY:
- Implement feature-specific post-processing

## Common Issues

- symptoms: "AnalyzeDocument returns no form data"
  diagnosis: "FORMS not included in FeatureTypes or document has no form structure."
  resolution: "Include FORMS in FeatureTypes. Verify document has key-value pairs."

- symptoms: "UnsupportedDocumentException"
  diagnosis: "Document format not supported or corrupted."
  resolution: "Convert to supported format. Check document integrity."

## Output Format

```yaml
root_cause: "document_analysis — <specific_cause>"
evidence:
  - type: analysis_result
    content: "<AnalyzeDocument response>"
  - type: feature_types
    content: "<requested feature types>"
severity: HIGH
mitigation:
  immediate: "Fix feature types or document format"
  long_term: "Implement document analysis pipeline with validation"
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
