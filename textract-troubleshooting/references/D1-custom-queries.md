---
title: "D1 — Custom Query Issues"
description: "Diagnose Textract Queries feature problems"
status: active
severity: MEDIUM
triggers:
  - "query not returning results"
  - "Queries feature error"
  - "query extraction wrong"
owner: devops-agent
objective: "Identify and resolve Queries feature issues"
context: "Queries allow natural language questions about documents. Requires QUERIES in FeatureTypes. Maximum 15 queries per call. Works best with structured documents."
---

## Phase 1 — Triage

MUST:
- Test with queries: `aws textract analyze-document --document '{"S3Object":{"Bucket":"<bucket>","Name":"<key>"}}' --feature-types '["QUERIES"]' --queries-config '{"Queries":[{"Text":"What is the total amount?"}]}'`
- Verify QUERIES is in FeatureTypes
- Check QUERY and QUERY_RESULT blocks in response
- Review confidence scores for query results

SHOULD:
- Test with different query phrasings
- Verify document contains the queried information
- Check query count (max 15 per call)

MAY:
- Compare query results with form extraction
- Test queries on different document types

## Phase 2 — Remediate

MUST:
- Include QUERIES in FeatureTypes
- Pass queries in QueriesConfig parameter
- Use clear, specific query text

SHOULD:
- Phrase queries to match document terminology
- Use aliases for alternative phrasings

MAY:
- Implement query optimization based on document type

## Common Issues

- symptoms: "Query returns empty result"
  diagnosis: "Query doesn't match document content or phrasing too different."
  resolution: "Rephrase query to match document terminology. Check document has the data."

- symptoms: "ValidationException for queries"
  diagnosis: "QUERIES not in FeatureTypes or exceeds 15 query limit."
  resolution: "Add QUERIES to FeatureTypes. Reduce to 15 or fewer queries."

## Output Format

```yaml
root_cause: "custom_queries — <specific_cause>"
evidence:
  - type: query_results
    content: "<QUERY_RESULT blocks>"
  - type: confidence
    content: "<query result confidence>"
severity: MEDIUM
mitigation:
  immediate: "Fix query configuration or phrasing"
  long_term: "Develop query templates per document type"
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
