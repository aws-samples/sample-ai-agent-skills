---
title: "E1 — Expense Analysis Issues"
description: "Diagnose expense document analysis problems"
status: active
severity: MEDIUM
triggers:
  - "AnalyzeExpense error"
  - "receipt extraction failed"
  - "invoice fields missing"
  - "expense analysis wrong"
owner: devops-agent
objective: "Identify and resolve expense analysis issues"
context: "AnalyzeExpense extracts standardized fields from receipts and invoices (vendor, total, line items, tax). Specialized API provides better results than generic AnalyzeDocument."
---

## Phase 1 — Triage

MUST:
- Test expense analysis: `aws textract analyze-expense --document '{"S3Object":{"Bucket":"<bucket>","Name":"<key>"}}'`
- Review extracted summary fields and line items
- Check confidence scores for key fields
- Verify document is a receipt or invoice

SHOULD:
- Check for missing standard fields (VENDOR_NAME, TOTAL, etc.)
- Verify line item extraction accuracy
- Test with different receipt/invoice formats

MAY:
- Compare with AnalyzeDocument FORMS results
- Check for multi-page invoice handling

## Phase 2 — Remediate

MUST:
- Use AnalyzeExpense (not AnalyzeDocument) for receipts/invoices
- Handle both summary fields and line items in parsing
- Validate extracted amounts and totals

SHOULD:
- Implement field-specific validation (date format, currency)
- Handle missing fields gracefully

MAY:
- Build expense extraction pipeline with validation

## Common Issues

- symptoms: "Total amount not extracted"
  diagnosis: "Document layout doesn't match expected receipt/invoice format."
  resolution: "Verify document is a receipt/invoice. Check for non-standard layouts."

- symptoms: "Line items missing or incomplete"
  diagnosis: "Complex line item layout or poor document quality."
  resolution: "Improve document quality. Implement line item validation."

## Output Format

```yaml
root_cause: "expense_analysis — <specific_cause>"
evidence:
  - type: expense_result
    content: "<AnalyzeExpense response>"
  - type: field_confidence
    content: "<field confidence scores>"
severity: MEDIUM
mitigation:
  immediate: "Fix document quality or parsing logic"
  long_term: "Implement expense extraction validation pipeline"
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
