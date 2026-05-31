---
title: "C2 — Export Output Issues"
description: "Diagnose export output format and content problems"
status: active
severity: MEDIUM
triggers: ["export empty", "wrong format", "missing resources"]
owner: devops-agent
objective: "Identify and resolve export output issues"
context: "Output issues occur when: filter too restrictive, time range wrong, resource type not specified, or NDJSON parsing fails."
---

## Phase 1 — Triage

MUST:
- Check export job configuration for filters
- Verify time range includes expected data
- Check resource type filter
- Verify output files in S3: `aws s3 ls s3://<bucket>/<prefix>/`

SHOULD:
- Check NDJSON output format
- Verify file sizes are reasonable
- Check for partial exports

## Phase 2 — Remediate

MUST:
- Adjust time range to include expected data
- Fix resource type filter
- Verify data exists in data store before exporting

SHOULD:
- Test with broad filters first, then narrow
- Document export configurations

## Output Format

```yaml
root_cause: "export_output — <specific_cause>"
evidence:
  - type: export_config
    content: "<filter configuration>"
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
