---
title: "E1 — Analytics Query Issues"
description: "Diagnose HealthLake analytics and Athena query problems"
status: active
severity: MEDIUM
triggers: ["Athena query failed", "analytics not available", "SQL error"]
owner: devops-agent
objective: "Identify and resolve analytics query issues"
context: "Analytics issues occur when: analytics not enabled, sync delay, Athena permissions wrong, or SQL syntax error."
---

## Phase 1 — Triage

MUST:
- Verify analytics is enabled on data store
- Check Lake Formation database exists
- Verify Athena permissions for the analytics database
- Check SQL syntax for Athena compatibility

SHOULD:
- Allow time for analytics data sync
- Check Athena query execution history for errors
- Verify table schema matches expected FHIR structure

## Phase 2 — Remediate

MUST:
- Enable analytics on data store if not enabled
- Grant Athena and Lake Formation permissions
- Fix SQL syntax

SHOULD:
- Allow sync time before querying new data
- Use Athena workgroups for cost control

## Output Format

```yaml
root_cause: "analytics_query — <specific_cause>"
evidence:
  - type: athena_error
    content: "<query error>"
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
