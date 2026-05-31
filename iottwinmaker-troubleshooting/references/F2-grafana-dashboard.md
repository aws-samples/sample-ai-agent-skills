---
title: "F2 — Grafana Dashboard Issues"
description: "Diagnose TwinMaker panels in Grafana dashboards"
status: active
severity: LOW
triggers: ["panel error", "scene not showing", "no data in panel"]
owner: devops-agent
objective: "Identify and resolve Grafana dashboard issues"
context: "Dashboard issues occur when: panel configuration wrong, entity/property references invalid, or scene panel not loading."
---

## Phase 1 — Triage

MUST:
- Check panel data source is set to TwinMaker
- Verify entity and property references in panel
- Check scene panel configuration for scene ID
- Test data source connection

SHOULD:
- Verify panel type matches use case (scene, property, alarm)
- Check time range settings
- Verify Grafana user has permissions

## Phase 2 — Remediate

MUST:
- Fix entity/property references in panel
- Correct scene ID in scene panel
- Fix data source selection

SHOULD:
- Test panels with known entities and properties
- Use Grafana query inspector to debug

## Output Format

```yaml
root_cause: "grafana_dashboard — <specific_cause>"
evidence:
  - type: panel_config
    content: "<panel configuration>"
severity: LOW
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
