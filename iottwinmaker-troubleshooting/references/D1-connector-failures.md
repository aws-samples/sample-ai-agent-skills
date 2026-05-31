---
title: "D1 — Data Connector Failures"
description: "Diagnose data connector configuration and execution failures"
status: active
severity: HIGH
triggers: ["connector error", "no data from connector", "data source failed"]
owner: devops-agent
objective: "Identify and resolve data connector failures"
context: "Connector failures occur when: Lambda function missing, permissions insufficient, data source unreachable, or response format wrong."
---

## Phase 1 — Triage

MUST:
- Check component type data connector configuration
- Verify Lambda function exists: `aws lambda get-function --function-name <name>`
- Check Lambda function logs: `aws logs filter-log-events --log-group-name /aws/lambda/<name> --start-time <epoch>`
- Verify Lambda has permissions for data source

SHOULD:
- Test Lambda function directly with sample TwinMaker request
- Verify response format matches TwinMaker expectations
- Check Lambda timeout settings

## Phase 2 — Remediate

MUST:
- Fix Lambda function errors
- Grant Lambda permissions for data source
- Ensure response format matches TwinMaker API contract

SHOULD:
- Implement error handling and logging in Lambda
- Set appropriate Lambda timeout

## Output Format

```yaml
root_cause: "connector_failure — <specific_cause>"
evidence:
  - type: lambda_logs
    content: "<function error>"
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
