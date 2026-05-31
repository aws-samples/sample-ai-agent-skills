---
title: "B1 — Entity Errors"
description: "Diagnose entity creation and management errors"
status: active
severity: MEDIUM
triggers: ["entity creation failed", "entity not found", "entity error"]
owner: devops-agent
objective: "Identify and resolve entity errors"
context: "Entity errors occur when: component type doesn't exist, parent entity invalid, property values wrong type, or entity limit reached."
---

## Phase 1 — Triage

MUST:
- Check entity: `aws iottwinmaker get-entity --workspace-id <id> --entity-id <entity-id>`
- Verify component types exist: `aws iottwinmaker list-component-types --workspace-id <id>`
- Check parent entity if hierarchical
- Verify property values match component type schema

SHOULD:
- Check entity status for errors
- Verify entity ID uniqueness
- Check workspace entity limits

## Phase 2 — Remediate

MUST:
- Create missing component types before entities
- Fix parent entity references
- Correct property value types

SHOULD:
- Validate entity definitions before creation
- Use consistent naming conventions

## Output Format

```yaml
root_cause: "entity_error — <specific_cause>"
evidence:
  - type: entity_details
    content: "<entity configuration>"
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
