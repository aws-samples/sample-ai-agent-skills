---
title: "B2 — Table Extraction Issues"
description: "Diagnose table extraction and cell parsing problems"
status: active
severity: HIGH
triggers:
  - "table not detected"
  - "table cells wrong"
  - "merged cells issue"
  - "table extraction error"
owner: devops-agent
objective: "Identify and resolve table extraction issues"
context: "Table extraction returns TABLE and CELL blocks with row/column indices. Merged cells use RowSpan/ColumnSpan. Complex layouts may not extract perfectly."
---

## Phase 1 — Triage

MUST:
- Test table extraction: `aws textract analyze-document --document '{"S3Object":{"Bucket":"<bucket>","Name":"<key>"}}' --feature-types '["TABLES"]'`
- Check for TABLE and CELL blocks in response
- Verify row/column indices are correct
- Check for merged cell handling (RowSpan, ColumnSpan)

SHOULD:
- Verify document has clear table borders/structure
- Check confidence scores for cells
- Test with different table layouts

MAY:
- Compare with manual table structure
- Check for nested tables

## Phase 2 — Remediate

MUST:
- Include TABLES in FeatureTypes
- Parse CELL blocks with row/column indices
- Handle merged cells (RowSpan > 1 or ColumnSpan > 1)

SHOULD:
- Implement table reconstruction logic
- Handle header rows separately

MAY:
- Implement table validation against expected schema

## Common Issues

- symptoms: "Table detected but cells misaligned"
  diagnosis: "Complex table layout or unclear borders."
  resolution: "Use documents with clear table borders. Post-process cell alignment."

- symptoms: "Merged cells split into multiple cells"
  diagnosis: "Textract may not detect all merged cells."
  resolution: "Check RowSpan/ColumnSpan. Implement merge detection in post-processing."

## Output Format

```yaml
root_cause: "table_extraction — <specific_cause>"
evidence:
  - type: table_blocks
    content: "<TABLE and CELL blocks>"
  - type: structure
    content: "<row/column structure>"
severity: HIGH
mitigation:
  immediate: "Fix table parsing logic"
  long_term: "Implement robust table reconstruction"
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
