---
title: "D2 — Lambda Data Connector Functions"
description: "Diagnose Lambda function issues in data connectors"
status: active
severity: MEDIUM
triggers: ["Lambda timeout", "Lambda error", "wrong data format"]
owner: devops-agent
objective: "Identify and resolve Lambda data connector function issues"
context: "Lambda issues occur when: function timeout, data source query slow, response format invalid, or memory insufficient."
---

## Phase 1 — Triage

MUST:
- Check Lambda function configuration: `aws lambda get-function-configuration --function-name <name>`
- Review function logs for errors
- Verify function timeout is sufficient for data source queries
- Check memory allocation

SHOULD:
- Test function with TwinMaker request format
- Verify data source connectivity from Lambda VPC
- Check for cold start issues

## Phase 2 — Remediate

MUST:
- Increase timeout if data source queries are slow
- Fix response format to match TwinMaker contract
- Increase memory if OOM errors

SHOULD:
- Implement connection pooling for data sources
- Add caching for frequently accessed data

## Output Format

```yaml
root_cause: "lambda_connector — <specific_cause>"
evidence:
  - type: lambda_config
    content: "<function configuration>"
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
- Never delete workspaces containing active digital twin entities
- Never modify component types that are referenced by production entities
- Never remove data connectors without confirming alternative data sources
- Never alter scene configurations used by active monitoring dashboards
- Never delete entity hierarchies without understanding downstream visualization impact

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous component type definitions if entity data binding breaks
- Revert scene configuration changes if 3D visualization stops rendering
- Re-enable previous data connector settings if twin data synchronization fails

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
