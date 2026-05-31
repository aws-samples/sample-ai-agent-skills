---
title: "C1 — Query Relevance Issues"
description: "Diagnose poor search relevance and missing results"
status: active
severity: MEDIUM
triggers:
  - "poor search results"
  - "relevant documents not returned"
  - "wrong ranking"
owner: devops-agent
objective: "Improve query relevance in Kendra"
context: "Relevance depends on document quality, metadata, field boosting, and content structure."
---

## Phase 1 — Triage

MUST:
- Test query: `aws kendra query --index-id <index-id> --query-text "<query>"`
- Check result count and relevance scores
- Verify documents are indexed (sync completed)
- Check if ACLs are filtering results

SHOULD:
- Test with different query formulations
- Check document metadata quality
- Review field boosting configuration
- Check for relevance tuning settings

MAY:
- Use featured results for known important documents
- Check thesaurus configuration
- Review query analytics

## Phase 2 — Remediate

MUST:
- Ensure documents are synced and indexed
- Add metadata to improve relevance signals
- Configure field boosting for important attributes

SHOULD:
- Use relevance feedback to train ranking
- Add featured results for key queries
- Configure thesaurus for domain terms

## Common Issues

- symptoms: "Query returns 0 results but documents exist"
  diagnosis: "ACLs filtering all results or sync not complete."
  resolution: "Check ACL config. Verify sync status. Test without user context."

## Output Format

```yaml
root_cause: "query_relevance — <specific_cause>"
evidence:
  - type: query_results
    content: "<query response details>"
severity: MEDIUM
mitigation:
  immediate: "Fix immediate relevance blockers"
  long_term: "Implement relevance tuning strategy"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never delete indexes that serve production search applications
- Never modify data source connectors during active sync operations
- Never remove FAQ entries without confirming impact on search quality
- Never alter access control configurations without security review
- Never delete experience builder applications used by end users

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous data source connector configuration if sync operations fail
- Revert index relevance tuning settings if search quality degrades
- Re-enable previous ACL configurations if access control breaks

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
