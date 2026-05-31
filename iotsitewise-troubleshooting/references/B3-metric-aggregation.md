---
title: "B3 — Metric Aggregation Issues"
description: "Diagnose metric computation and aggregation problems"
status: active
severity: MEDIUM
triggers:
  - "metric not computing"
  - "aggregation wrong"
  - "metric delayed"
  - "metric value unexpected"
owner: devops-agent
objective: "Identify and resolve metric aggregation issues"
context: "Metric issues occur when: time window not elapsed, no input data in window, late data not recomputed, or aggregation function misunderstood."
---

## Phase 1 — Triage

MUST:
- Check metric value: `aws iotsitewise get-asset-property-value --asset-id <id> --property-id <metric-prop-id>`
- Check metric history: `aws iotsitewise batch-get-asset-property-value-history --entries '[{"assetId":"<id>","propertyId":"<prop-id>","entryId":"1","startDate":"<start>","endDate":"<end>"}]'`
- Verify input data exists within the time window
- Check time window configuration in model

SHOULD:
- Verify aggregation function (avg, sum, min, max, count) is correct
- Check if late-arriving data triggered recomputation
- Verify hierarchy aggregation if metric spans child assets

MAY:
- Compare metric result with manual calculation from raw data
- Check processing delay for recent time windows

## Phase 2 — Remediate

MUST:
- Ensure input data is being ingested within time windows
- Wait for time window to close before expecting results
- Fix aggregation function if wrong

SHOULD:
- Account for processing delay in monitoring
- Use appropriate time windows for data frequency

## Output Format

```yaml
root_cause: "metric_aggregation — <specific_cause>"
evidence:
  - type: metric_history
    content: "<metric values and time windows>"
severity: MEDIUM
mitigation:
  immediate: "Verify input data and time windows"
  long_term: "Monitor metric computation health"
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
