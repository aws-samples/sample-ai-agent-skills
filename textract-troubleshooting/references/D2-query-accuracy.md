---
title: "D2 — Query Accuracy Issues"
description: "Diagnose query result accuracy and relevance problems"
status: active
severity: MEDIUM
triggers:
  - "query returns wrong value"
  - "query low confidence"
  - "query inaccurate"
owner: devops-agent
objective: "Improve query extraction accuracy"
context: "Query accuracy depends on document quality, query phrasing, and document structure. Confidence scores indicate reliability. Multiple phrasings can improve results."
---

## Phase 1 — Triage

MUST:
- Review query result confidence scores
- Compare extracted value with actual document content
- Test with alternative query phrasings
- Check document quality and readability

SHOULD:
- Use query aliases for alternative phrasings
- Test with different MinConfidence thresholds
- Verify document structure matches query expectations

MAY:
- Compare with form extraction results
- Test across similar document types

## Phase 2 — Remediate

MUST:
- Use specific, document-aligned query text
- Set appropriate confidence thresholds
- Validate results against expected format

SHOULD:
- Use aliases for query variations
- Implement result validation logic

MAY:
- Build query accuracy benchmarks per document type

## Common Issues

- symptoms: "Query returns adjacent field value instead of target"
  diagnosis: "Query phrasing ambiguous or document layout confusing."
  resolution: "Make query more specific. Include context in query text."

- symptoms: "Low confidence on all query results"
  diagnosis: "Poor document quality or queries don't match document structure."
  resolution: "Improve document quality. Align queries with document terminology."

## Output Format

```yaml
root_cause: "query_accuracy — <specific_cause>"
evidence:
  - type: query_results
    content: "<query results with confidence>"
  - type: document_quality
    content: "<document quality assessment>"
severity: MEDIUM
mitigation:
  immediate: "Improve query phrasing"
  long_term: "Build query templates with accuracy benchmarks"
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
