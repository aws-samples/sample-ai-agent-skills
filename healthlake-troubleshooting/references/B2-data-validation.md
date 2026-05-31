---
title: "B2 — Data Validation Issues"
description: "Diagnose FHIR resource validation failures during import"
status: active
severity: MEDIUM
triggers: ["validation error", "invalid resource", "schema error"]
owner: devops-agent
objective: "Identify and resolve FHIR resource validation failures"
context: "Validation fails when: required fields missing, data types wrong, reference targets invalid, or resource type unsupported."
---

## Phase 1 — Triage

MUST:
- Check import job output manifest for validation errors
- Verify resource type is supported FHIR R4 type
- Check required fields are present (resourceType, id)
- Verify data types match FHIR R4 specification

SHOULD:
- Validate resources with FHIR R4 validator before import
- Check reference targets exist or will be created in same import
- Verify coding systems and value sets

## Phase 2 — Remediate

MUST:
- Fix missing required fields
- Correct data type mismatches
- Use valid FHIR R4 resource types

SHOULD:
- Implement pre-import validation pipeline
- Document common validation errors

## Output Format

```yaml
root_cause: "data_validation — <specific_cause>"
evidence:
  - type: validation_errors
    content: "<specific validation failures>"
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
