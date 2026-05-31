---
title: "A1 — Channel Ingestion Failures"
description: "Diagnose message ingestion failures into channels"
status: active
severity: HIGH
triggers:
  - "messages not arriving"
  - "channel empty"
  - "ingestion error"
  - "BatchPutMessage failed"
owner: devops-agent
objective: "Identify and resolve channel ingestion failures"
context: "Ingestion fails when: IoT Core rule misconfigured, BatchPutMessage permissions missing, message too large, or channel doesn't exist."
---

## Phase 1 — Triage

MUST:
- Verify channel exists: `aws iotanalytics describe-channel --channel-name <name>`
- Check IoT Core rule action if routing from IoT Core: `aws iot get-topic-rule --rule-name <name>`
- Check CloudWatch ingestion metrics: `aws cloudwatch get-metric-statistics --namespace AWS/IoTAnalytics --metric-name IncomingMessages --dimensions Name=ChannelName,Value=<name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify IAM permissions for ingestion

SHOULD:
- Sample channel data: `aws iotanalytics sample-channel-data --channel-name <name>`
- Check message size (max 128 KB)
- Verify message format is valid JSON

MAY:
- Test with BatchPutMessage: `aws iotanalytics batch-put-message --channel-name <name> --messages '[{"messageId":"test1","payload":"<base64>"}]'`

## Phase 2 — Remediate

MUST:
- Fix IoT Core rule action configuration
- Grant iotanalytics:BatchPutMessage permission
- Ensure messages are under 128 KB

SHOULD:
- Monitor ingestion success rates
- Implement error handling for batch operations

## Output Format

```yaml
root_cause: "ingestion_failure — <specific_cause>"
evidence:
  - type: channel_metrics
    content: "<ingestion counts>"
severity: HIGH
mitigation:
  immediate: "Fix rule action or permissions"
  long_term: "Monitor ingestion health"
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
- Never delete channels or datastores that are actively receiving data
- Never modify pipeline activities on production pipelines without testing
- Never change data retention settings without understanding downstream impact
- Never delete datasets that feed dashboards or reporting systems
- Never alter SQL query definitions in scheduled datasets without validation

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous pipeline activity configuration if data processing breaks
- Revert channel or datastore settings if data ingestion stops
- Re-enable previous dataset scheduling if automated analysis is disrupted

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
