---
title: "E2 — Lake Formation Integration Issues"
description: "Diagnose Lake Formation integration problems"
status: active
severity: MEDIUM
triggers: ["Lake Formation error", "database not found", "permission denied"]
owner: devops-agent
objective: "Identify and resolve Lake Formation integration issues"
context: "Integration issues occur when: Lake Formation not configured, permissions missing, or database sync failed."
---

## Phase 1 — Triage

MUST:
- Verify Lake Formation database exists for the data store
- Check Lake Formation permissions for the querying identity
- Verify analytics integration status on data store
- Check for sync errors

SHOULD:
- Verify Lake Formation data location permissions
- Check if tables are populated with data
- Review Lake Formation audit logs

## Phase 2 — Remediate

MUST:
- Grant Lake Formation permissions to querying identity
- Fix data location permissions
- Re-enable analytics if sync failed

SHOULD:
- Use Lake Formation fine-grained access control
- Monitor sync status

## Output Format

```yaml
root_cause: "lake_formation — <specific_cause>"
evidence:
  - type: lf_permissions
    content: "<permission details>"
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
