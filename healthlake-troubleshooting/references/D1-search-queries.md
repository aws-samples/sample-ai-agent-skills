---
title: "D1 — Search Query Issues"
description: "Diagnose FHIR search query failures"
status: active
severity: MEDIUM
triggers: ["search error", "no results", "search parameter invalid"]
owner: devops-agent
objective: "Identify and resolve FHIR search query failures"
context: "Search fails when: search parameter unsupported, syntax wrong, resource type invalid, or data not yet indexed."
---

## Phase 1 — Triage

MUST:
- Verify search parameter is supported for the resource type
- Check FHIR search syntax (parameter=value format)
- Verify resource type exists in data store
- Check if recently imported data is indexed

SHOULD:
- Test with simple search first (e.g., GET /Patient)
- Check pagination parameters (_count, _offset)
- Verify search modifiers (:exact, :contains, :missing)

## Phase 2 — Remediate

MUST:
- Use supported FHIR R4 search parameters
- Fix search syntax
- Allow time for indexing after import

SHOULD:
- Use _include/_revinclude for related resources
- Implement pagination for large result sets

## Output Format

```yaml
root_cause: "search_query — <specific_cause>"
evidence:
  - type: search_request
    content: "<query and response>"
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
