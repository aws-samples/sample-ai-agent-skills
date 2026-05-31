---
title: "B1 — Measurement Ingestion Issues"
description: "Diagnose measurement data ingestion failures"
status: active
severity: HIGH
triggers:
  - "data not appearing"
  - "measurement missing"
  - "ingestion failed"
  - "BatchPutAssetPropertyValue error"
owner: devops-agent
objective: "Identify and resolve measurement data ingestion failures"
context: "Ingestion fails when: data type mismatch, timestamp out of range, rate limit exceeded, property alias wrong, or permissions insufficient."
---

## Phase 1 — Triage

MUST:
- Check property value: `aws iotsitewise get-asset-property-value --asset-id <id> --property-id <prop-id>`
- Check ingestion metrics: `aws cloudwatch get-metric-statistics --namespace AWS/IoTSiteWise --metric-name BatchPutAssetPropertyValue.Success --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify property data type matches ingested data
- Check timestamp is within allowed range

SHOULD:
- Verify property alias if using alias-based ingestion
- Check IAM permissions for iotsitewise:BatchPutAssetPropertyValue
- Review error metrics for specific failure reasons

MAY:
- Test ingestion with a single value: `aws iotsitewise batch-put-asset-property-value --entries '[{"entryId":"test","assetId":"<id>","propertyId":"<prop-id>","propertyValues":[{"value":{"doubleValue":25.0},"timestamp":{"timeInSeconds":<epoch>}}]}]'`

## Phase 2 — Remediate

MUST:
- Fix data type to match model definition
- Ensure timestamps are within ingestion window
- Grant required IAM permissions

SHOULD:
- Implement ingestion error handling and retry
- Monitor ingestion success/failure rates

## Output Format

```yaml
root_cause: "measurement_ingestion — <specific_cause>"
evidence:
  - type: ingestion_metrics
    content: "<success/failure counts>"
severity: HIGH
mitigation:
  immediate: "Fix data type or timestamp"
  long_term: "Implement ingestion monitoring"
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
