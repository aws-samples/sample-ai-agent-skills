---
title: "G2 — Access Control Issues"
description: "Diagnose IAM and SMART on FHIR access control problems"
status: active
severity: HIGH
triggers: ["access denied", "unauthorized", "SMART auth error"]
owner: devops-agent
objective: "Identify and resolve access control issues"
context: "Access issues occur when: IAM policy missing, SMART on FHIR misconfigured, or cross-account access not set up."
---

## Phase 1 — Triage

MUST:
- Check IAM policy for healthlake:* permissions
- Verify resource ARN in IAM policy matches data store
- Check if SMART on FHIR is configured
- Review CloudTrail for access denied events

SHOULD:
- Verify SMART on FHIR authorization server configuration
- Check token validation settings
- Verify cross-account role if applicable

## Phase 2 — Remediate

MUST:
- Grant required IAM permissions
- Fix resource ARN in IAM policy
- Configure SMART on FHIR authorization correctly

SHOULD:
- Use least-privilege IAM policies
- Implement SMART on FHIR for patient-level access control

## Output Format

```yaml
root_cause: "access_control — <specific_cause>"
evidence:
  - type: iam_policy
    content: "<policy details>"
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
