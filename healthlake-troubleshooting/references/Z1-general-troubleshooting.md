---
title: "Z1 — General HealthLake Troubleshooting"
description: "Catch-all runbook for unclassified HealthLake issues"
status: active
severity: MEDIUM
triggers: ["HealthLake not working", "something wrong with HealthLake"]
owner: devops-agent
objective: "Systematically triage unclassified HealthLake issues"
---

## Phase 1 — Triage

MUST:
- List data stores: `aws healthlake list-fhir-datastores`
- Check data store status: `aws healthlake describe-fhir-datastore --datastore-id <id>`
- Check recent import/export jobs
- Review CloudTrail for errors

SHOULD:
- Check IAM permissions
- Verify service health
- Check CloudWatch metrics

## Phase 2 — Route to specific runbook

Based on findings, route to A1-A2, B1-B2, C1-C2, D1-D2, E1-E2, F1-F2, or G1-G2.

## Output Format

```yaml
root_cause: "general — <routed_to_specific_runbook>"
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
