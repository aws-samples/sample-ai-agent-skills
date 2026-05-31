---
title: "C1 — Export Failures"
description: "Diagnose FHIR data export job failures"
status: active
severity: HIGH
triggers: ["export failed", "export error", "S3 write failed"]
owner: devops-agent
objective: "Identify and resolve FHIR export failures"
context: "Export fails when: S3 permissions wrong, bucket doesn't exist, KMS key blocks write, or data store not active."
---

## Phase 1 — Triage

MUST:
- Check export job status: `aws healthlake describe-fhir-export-job --datastore-id <id> --job-id <job-id>`
- Verify S3 output bucket exists
- Check IAM role has s3:PutObject on output bucket
- Verify data store is in ACTIVE state

SHOULD:
- Check KMS key permissions on output bucket
- Verify export filter configuration (resource types, time range)
- Check for concurrent export jobs

## Phase 2 — Remediate

MUST:
- Fix S3 bucket permissions
- Create output bucket if missing
- Fix KMS key policy

SHOULD:
- Use dedicated S3 prefix for exports
- Monitor export job progress

## Output Format

```yaml
root_cause: "export_failure — <specific_cause>"
evidence:
  - type: export_status
    content: "<job details>"
severity: HIGH
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
