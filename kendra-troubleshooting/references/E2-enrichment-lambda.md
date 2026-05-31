---
title: "E2 — Enrichment Lambda Errors"
description: "Diagnose Lambda function errors in document enrichment"
status: active
severity: MEDIUM
triggers:
  - "Lambda enrichment failed"
  - "enrichment timeout"
  - "Lambda error during sync"
owner: devops-agent
objective: "Resolve Lambda-based enrichment errors"
context: "CDE Lambda hooks run during ingestion. Lambda errors can block document processing."
---

## Phase 1 — Triage

MUST:
- Check Lambda function exists and is accessible
- Review Lambda CloudWatch Logs for errors
- Verify Lambda IAM role permissions
- Check Lambda timeout configuration

SHOULD:
- Verify Lambda response format matches Kendra expectations
- Check for Lambda concurrency limits
- Test Lambda independently with sample input

MAY:
- Check Lambda memory allocation
- Review Lambda error handling

## Phase 2 — Remediate

MUST:
- Fix Lambda code errors
- Increase timeout if timing out
- Fix IAM permissions

SHOULD:
- Add error handling to Lambda
- Configure dead letter queue for failed invocations

## Common Issues

- symptoms: "Sync fails with Lambda timeout"
  diagnosis: "Lambda processing takes too long per document."
  resolution: "Increase Lambda timeout. Optimize processing logic."

## Output Format

```yaml
root_cause: "enrichment_lambda — <specific_cause>"
evidence:
  - type: lambda_logs
    content: "<Lambda error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix Lambda errors"
  long_term: "Add monitoring and error handling"
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
