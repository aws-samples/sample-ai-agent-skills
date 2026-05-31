---
title: "B3 — Property Value Issues"
description: "Diagnose property value retrieval and update failures"
status: active
severity: MEDIUM
triggers: ["property value error", "no data returned", "data connector timeout"]
owner: devops-agent
objective: "Identify and resolve property value issues"
context: "Property value issues occur when: data connector Lambda fails, permissions missing, property not defined, or value type mismatch."
---

## Phase 1 — Triage

MUST:
- Get property value: `aws iottwinmaker get-property-value --workspace-id <id> --entity-id <entity-id> --component-name <name> --selected-properties '["<prop>"]'`
- Check data connector Lambda function logs
- Verify property is defined in component type
- Check Lambda function permissions

SHOULD:
- Test Lambda function directly with sample input
- Verify data source accessibility from Lambda
- Check for Lambda timeout issues

## Phase 2 — Remediate

MUST:
- Fix Lambda function errors
- Grant Lambda permissions for data source
- Correct property name references

SHOULD:
- Implement caching in Lambda for frequently accessed values
- Add error handling and logging

## Output Format

```yaml
root_cause: "property_value — <specific_cause>"
evidence:
  - type: property_response
    content: "<error or empty response>"
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
