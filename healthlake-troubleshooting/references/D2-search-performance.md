---
title: "D2 — Search Performance Issues"
description: "Diagnose FHIR search performance problems"
status: active
severity: MEDIUM
triggers: ["slow search", "search timeout", "high latency"]
owner: devops-agent
objective: "Identify and resolve search performance issues"
context: "Performance issues occur when: broad queries, too many includes, large result sets, or complex chained searches."
---

## Phase 1 — Triage

MUST:
- Check query complexity (number of parameters, includes)
- Verify _count parameter limits result size
- Check if query uses efficient search parameters
- Monitor CloudWatch for search latency metrics

SHOULD:
- Test with simpler queries to establish baseline
- Check if _include/_revinclude is causing large result sets
- Verify chained search parameters are indexed

## Phase 2 — Remediate

MUST:
- Add _count parameter to limit results
- Simplify complex queries
- Use specific search parameters instead of broad searches

SHOULD:
- Implement pagination for large result sets
- Cache frequently accessed data

## Output Format

```yaml
root_cause: "search_performance — <specific_cause>"
evidence:
  - type: query_metrics
    content: "<latency and result size>"
severity: MEDIUM
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
- Never delete FHIR datastores containing patient health data without authorization
- Never modify encryption settings on active datastores without compliance review
- Never import untested or unvalidated FHIR data into production datastores
- Never disable access controls or audit logging on datastores with PHI
- Never export patient data without verifying HIPAA compliance requirements

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous datastore configuration from documented state if imports fail
- Revert access control changes immediately if unauthorized access is detected
- Re-enable encryption and audit settings if they were inadvertently modified

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
