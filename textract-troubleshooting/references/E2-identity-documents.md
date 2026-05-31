---
title: "E2 — Identity Document Analysis Issues"
description: "Diagnose identity document analysis problems"
status: active
severity: MEDIUM
triggers:
  - "AnalyzeID error"
  - "ID extraction failed"
  - "passport analysis error"
  - "driver license extraction"
owner: devops-agent
objective: "Identify and resolve identity document analysis issues"
context: "AnalyzeID extracts standardized fields from identity documents (driver's licenses, passports). Returns fields like name, DOB, document number. Supports US driver's licenses and international passports."
---

## Phase 1 — Triage

MUST:
- Test ID analysis: `aws textract analyze-id --document-pages '[{"S3Object":{"Bucket":"<bucket>","Name":"<key>"}}]'`
- Review extracted identity fields
- Check confidence scores for key fields
- Verify document type is supported

SHOULD:
- Check for both front and back of ID (if applicable)
- Verify image quality and clarity
- Test with different ID types

MAY:
- Compare with generic AnalyzeDocument results
- Check for MRZ (Machine Readable Zone) extraction

## Phase 2 — Remediate

MUST:
- Use AnalyzeID (not AnalyzeDocument) for identity documents
- Provide clear, high-resolution images
- Handle both front and back pages

SHOULD:
- Validate extracted fields (date format, document number format)
- Implement confidence-based review workflow

MAY:
- Build ID verification pipeline

## Common Issues

- symptoms: "AnalyzeID returns few fields"
  diagnosis: "Image quality too low or unsupported ID type."
  resolution: "Use higher resolution image. Verify ID type is supported."

- symptoms: "Fields extracted with low confidence"
  diagnosis: "Poor image quality, glare, or partial document."
  resolution: "Capture full document without glare. Improve lighting."

## Output Format

```yaml
root_cause: "identity_documents — <specific_cause>"
evidence:
  - type: id_result
    content: "<AnalyzeID response>"
  - type: field_confidence
    content: "<field confidence scores>"
severity: MEDIUM
mitigation:
  immediate: "Improve image quality"
  long_term: "Implement ID capture guidelines and validation"
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
