---
title: "B1 — Form Extraction Issues"
description: "Diagnose form key-value pair extraction problems"
status: active
severity: HIGH
triggers:
  - "form extraction failed"
  - "key-value pairs missing"
  - "form fields not detected"
owner: devops-agent
objective: "Identify and resolve form extraction issues"
context: "Form extraction returns KEY_VALUE_SET blocks linked by relationships. Keys and values are separate blocks connected by IDs. Works best with structured forms."
---

## Phase 1 — Triage

MUST:
- Test form extraction: `aws textract analyze-document --document '{"S3Object":{"Bucket":"<bucket>","Name":"<key>"}}' --feature-types '["FORMS"]'`
- Check for KEY_VALUE_SET blocks in response
- Verify key-value relationships are present
- Check confidence scores for extracted pairs

SHOULD:
- Verify document has clear form structure
- Check for handwritten vs printed content
- Test with different form layouts

MAY:
- Compare with Queries feature for specific fields
- Check for checkbox/selection detection

## Phase 2 — Remediate

MUST:
- Include FORMS in FeatureTypes
- Parse KEY_VALUE_SET blocks and follow relationships
- Handle missing values (key without value)

SHOULD:
- Use confidence thresholds to filter unreliable extractions
- Implement key name normalization for consistency

MAY:
- Use Queries for specific field extraction as fallback

## Common Issues

- symptoms: "Key detected but value is empty"
  diagnosis: "Value field is blank in document or not adjacent to key."
  resolution: "Check document layout. Use Queries for non-adjacent key-value pairs."

- symptoms: "Form fields not detected as key-value pairs"
  diagnosis: "Document layout doesn't match expected form structure."
  resolution: "Use Queries feature for specific field extraction."

## Output Format

```yaml
root_cause: "form_extraction — <specific_cause>"
evidence:
  - type: form_blocks
    content: "<KEY_VALUE_SET blocks>"
  - type: confidence
    content: "<confidence scores>"
severity: HIGH
mitigation:
  immediate: "Fix feature types or use Queries"
  long_term: "Implement form extraction with validation and fallback"
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
