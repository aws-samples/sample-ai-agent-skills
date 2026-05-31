---
title: "F1 — Grafana Plugin Setup Issues"
description: "Diagnose TwinMaker Grafana plugin installation and configuration"
status: active
severity: MEDIUM
triggers: ["Grafana plugin error", "TwinMaker data source failed", "plugin not found"]
owner: devops-agent
objective: "Identify and resolve Grafana plugin setup issues"
context: "Setup issues occur when: plugin not installed, IAM auth misconfigured, workspace ID wrong, or region mismatch."
---

## Phase 1 — Triage

MUST:
- Verify TwinMaker plugin is installed in Grafana
- Check data source configuration (workspace ID, region)
- Verify IAM authentication (role ARN, assume role)
- Test data source connection in Grafana

SHOULD:
- Check Grafana version compatibility
- Verify Grafana workspace IAM role has iottwinmaker permissions
- Check network connectivity from Grafana to TwinMaker

## Phase 2 — Remediate

MUST:
- Install TwinMaker plugin if missing
- Fix data source configuration (workspace ID, region)
- Configure IAM authentication correctly

SHOULD:
- Test with simple property query first
- Document Grafana configuration

## Output Format

```yaml
root_cause: "grafana_setup — <specific_cause>"
evidence:
  - type: grafana_config
    content: "<data source configuration>"
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
