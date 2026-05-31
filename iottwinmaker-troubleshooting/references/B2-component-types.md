---
title: "B2 — Component Type Issues"
description: "Diagnose component type definition and validation errors"
status: active
severity: MEDIUM
triggers: ["component type error", "property definition invalid", "schema error"]
owner: devops-agent
objective: "Identify and resolve component type issues"
context: "Component type issues occur when: property definitions invalid, data connector Lambda missing, or inherited type conflicts."
---

## Phase 1 — Triage

MUST:
- Check component type: `aws iottwinmaker get-component-type --workspace-id <id> --component-type-id <type-id>`
- Verify property definitions (name, type, data connector)
- Check Lambda function for data connector properties
- Verify inherited component types exist

SHOULD:
- Check if component type is used by entities
- Verify Lambda function permissions
- Check for property name conflicts

## Phase 2 — Remediate

MUST:
- Fix property definitions
- Create or fix Lambda data connector functions
- Resolve inheritance conflicts

SHOULD:
- Test component types with sample entities
- Document component type schemas

## Output Format

```yaml
root_cause: "component_type — <specific_cause>"
evidence:
  - type: component_type_def
    content: "<type definition>"
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
