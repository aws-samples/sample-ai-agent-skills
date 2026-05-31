---
title: "F1 — Plugin Availability Issues"
description: "Diagnose plugin availability problems in Amazon Managed Grafana"
status: active
severity: MEDIUM
triggers:
  - "plugin not available"
  - "plugin not found"
  - "missing plugin"
  - "install plugin"
  - "community plugin"
owner: devops-agent
objective: "Identify why a required plugin is not available in AMG and find alternatives"
context: "AMG only supports plugins from the AWS-approved plugin catalog. Community plugins from the Grafana registry cannot be installed. Plugin availability depends on the workspace version and the AWS-managed catalog. Users migrating from self-hosted Grafana may encounter missing plugins."
---

## Phase 1 — Triage

MUST:
- Check available plugins on the workspace: `aws grafana describe-workspace-configuration --workspace-id <workspace-id>`
- Identify the required plugin name and type (panel, data source, app)
- Verify if the plugin is in the AMG-approved catalog
- Check the workspace Grafana version: `aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.grafanaVersion'`

SHOULD:
- Check if the plugin is available in a newer Grafana version
- Review the Grafana UI Plugins section for available plugins
- Identify alternative plugins that provide similar functionality

MAY:
- Check AWS documentation for the current AMG plugin catalog
- Review the dashboard or feature that requires the plugin for alternative approaches

## Phase 2 — Remediate

MUST:
- If the plugin is in the AMG catalog, enable it: `aws grafana update-workspace-configuration --workspace-id <workspace-id> --configuration '{"plugins":{"pluginAdminEnabled":true}}'`
- If the plugin is not available, identify an alternative from the AMG catalog

SHOULD:
- Replace unsupported panel plugins with built-in Grafana panel types (Time series, Stat, Table, etc.)
- Replace unsupported data source plugins with native AMG data sources where possible
- Document plugin dependencies for dashboards shared across environments

MAY:
- Request plugin addition through AWS Support if it's a commonly needed plugin
- Redesign dashboards to use only AMG-supported plugins for portability

## Common Issues

- symptoms: "Plugin 'grafana-piechart-panel' not found"
  diagnosis: "The pie chart panel was a community plugin in older Grafana versions. In newer versions, it's built-in."
  resolution: "Use the built-in Pie Chart panel type available in Grafana 8+. Update the dashboard JSON panel type."

- symptoms: "Cannot install community plugin via grafana-cli"
  diagnosis: "AMG does not support grafana-cli or manual plugin installation. Only AWS-approved plugins are available."
  resolution: "Check the AMG plugin catalog for the plugin. If not available, use an alternative or built-in panel type."

- symptoms: "Plugin available in self-hosted Grafana but not in AMG"
  diagnosis: "The plugin is not in the AWS-approved catalog for AMG."
  resolution: "Find an alternative plugin from the AMG catalog or redesign the visualization using built-in panel types."

## Safety Ratings

```
safety_ratings:
  - "describe-workspace-configuration: GREEN — read-only plugin check"
  - "check Grafana UI Plugins section: GREEN — read-only enumeration"
  - "update-workspace-configuration (enable plugin): YELLOW — changes workspace capabilities"
  - "replace panel types in dashboard JSON: YELLOW — modifies dashboard configuration"
```

## Escalation Conditions

- Critical dashboard depends on a plugin not available in the AMG catalog
- Migration from self-hosted Grafana blocked by missing plugins
- Plugin availability differs between Grafana versions, requiring upgrade planning
- Multiple dashboards across teams depend on the same unavailable plugin

## Data Sensitivity

- Plugin catalog: LOW — publicly available information
- Dashboard plugin dependencies: LOW — reveals visualization types used
- Workspace Grafana version: LOW — workspace metadata

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER attempt to install plugins via grafana-cli (not supported in AMG)
- NEVER modify the AMG plugin catalog directly (AWS-managed)

## Phase 3 — Rollback

1. If a plugin was enabled and caused issues, disable it via workspace configuration
2. If dashboard panels were changed to alternative types, restore from JSON backup
3. If workspace configuration was updated, restore previous configuration: `aws grafana update-workspace-configuration --workspace-id <id> --configuration <original-config>`
4. Verify dashboards render correctly after rollback
5. Document plugin dependencies for future migration planning

## Output Format

```yaml
root_cause: "plugin_availability — <specific_cause>"
evidence:
  - type: workspace_plugins
    content: "<available plugins in workspace>"
  - type: required_plugin
    content: "<plugin name and type needed>"
  - type: grafana_version
    content: "<workspace Grafana version>"
severity: MEDIUM
mitigation:
  immediate: "Enable the plugin if available, or use an alternative"
  long_term: "Standardize dashboards on AMG-supported plugins only"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
