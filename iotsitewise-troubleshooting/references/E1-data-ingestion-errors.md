---
title: "E1 — Data Ingestion Errors"
description: "Diagnose data ingestion API errors and throttling"
status: active
severity: HIGH
triggers:
  - "throttling"
  - "rate exceeded"
  - "ingestion error"
  - "BatchPutAssetPropertyValue failed"
owner: devops-agent
objective: "Identify and resolve data ingestion errors"
context: "Ingestion errors occur when: rate limits exceeded, payload invalid, permissions missing, or service unavailable."
---

## Phase 1 — Triage

MUST:
- Check ingestion metrics: `aws cloudwatch get-metric-statistics --namespace AWS/IoTSiteWise --metric-name BatchPutAssetPropertyValue.Error --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check throttling metrics for rate limit issues
- Verify IAM permissions for ingestion API
- Validate payload format (data types, timestamps, entry limits)

SHOULD:
- Check per-property ingestion rate
- Verify account-level ingestion limits
- Review error responses for specific error codes

MAY:
- Check Service Quotas for current limits: `aws service-quotas get-service-quota --service-code iotsitewise --quota-code <code>`

## Phase 2 — Remediate

MUST:
- Implement exponential backoff for throttling
- Fix payload validation errors
- Grant required IAM permissions

SHOULD:
- Batch data efficiently (up to 10 entries per call)
- Request quota increases if consistently hitting limits

## Output Format

```yaml
root_cause: "ingestion_error — <specific_cause>"
evidence:
  - type: ingestion_metrics
    content: "<error and throttling counts>"
severity: HIGH
mitigation:
  immediate: "Fix payload or implement backoff"
  long_term: "Optimize ingestion rate and request quota increases"
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
- Never delete asset models that have active assets associated with them
- Never modify measurement properties on assets receiving live industrial data
- Never remove gateway configurations without confirming data source migration
- Never alter transform or metric computations on production assets without testing
- Never delete portal or dashboard configurations used by operations teams

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous asset model configuration if property computations break
- Revert gateway data source settings if industrial data ingestion stops
- Re-enable previous alarm configurations if monitoring coverage is lost

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
