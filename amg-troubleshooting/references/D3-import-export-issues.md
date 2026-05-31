---
title: "D3 — Dashboard Import/Export Issues"
description: "Diagnose dashboard import and export problems in Amazon Managed Grafana"
status: active
severity: LOW
triggers:
  - "dashboard import"
  - "dashboard export"
  - "import failed"
  - "JSON import"
  - "migrate dashboard"
owner: devops-agent
objective: "Identify why dashboard import or export fails in AMG and complete the migration"
context: "AMG supports Grafana dashboard JSON import/export. However, imported dashboards may fail if they reference plugins not available in AMG, use data source types not enabled on the workspace, or contain version-incompatible configurations. Dashboards from self-hosted Grafana may need adaptation."
---

## Phase 1 — Triage

MUST:
- Check the import error message in the Grafana UI
- Review the dashboard JSON for plugin dependencies (panel types, data source types)
- Verify all referenced plugins are available in the AMG workspace: `aws grafana describe-workspace-configuration --workspace-id <workspace-id>`
- Check if referenced data source types are enabled on the workspace

SHOULD:
- Verify data source UIDs in the JSON match existing data sources in the workspace
- Check the Grafana version compatibility between source and target
- Look for deprecated panel types or query formats in the JSON

MAY:
- Compare the dashboard JSON schema with the target Grafana version requirements
- Check for custom variable queries that may not work in AMG

## Phase 2 — Remediate

MUST:
- Replace unsupported plugin references with available alternatives in the JSON
- Update data source UIDs to match the target workspace data sources
- Enable required data source types on the workspace before importing

SHOULD:
- Use the Grafana API for programmatic import: `curl -X POST -H "Authorization: Bearer <api-key>" -H "Content-Type: application/json" -d @dashboard.json https://<workspace-endpoint>/api/dashboards/db`
- Update deprecated panel types to current equivalents
- Test the dashboard after import to verify all panels render correctly

MAY:
- Create a migration script to batch-update data source UIDs across multiple dashboards
- Document plugin dependencies for dashboards shared across environments

## Common Issues

- symptoms: "Import fails with 'plugin not found' error"
  diagnosis: "Dashboard JSON references a plugin not available in the AMG plugin catalog."
  resolution: "Edit the JSON to replace the unsupported plugin with an available alternative (e.g., replace a community panel with a built-in panel type)."

- symptoms: "Dashboard imports but panels show 'Datasource not found'"
  diagnosis: "Data source UIDs in the JSON do not match any data source in the target workspace."
  resolution: "Update the data source UIDs in the JSON to match the target workspace data sources, or select data sources during import."

- symptoms: "Dashboard imports but variables do not populate"
  diagnosis: "Variable queries reference data sources or metrics not available in the target workspace."
  resolution: "Update variable queries to use available data sources and valid metric names."

## Safety Ratings

```
safety_ratings:
  - "check dashboard JSON for dependencies: GREEN — read-only analysis"
  - "describe-workspace-configuration: GREEN — read-only plugin check"
  - "import dashboard via API: YELLOW — creates or overwrites dashboard"
  - "update data source UIDs in JSON: YELLOW — modifies dashboard configuration"
```

## Escalation Conditions

- Migration of critical dashboards from self-hosted Grafana to AMG
- Dashboard import fails due to unsupported plugins with no clear alternative
- Bulk migration of many dashboards requires automated tooling
- Imported dashboards reference data sources in other accounts

## Data Sensitivity

- Dashboard JSON: MEDIUM — may contain query patterns and data source references
- Plugin dependencies: LOW — reveals which plugins are used
- Data source UIDs: MEDIUM — reveals data source configuration identifiers

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER overwrite existing production dashboards during import without backup
- NEVER import dashboards with hardcoded credentials in JSON

## Phase 3 — Rollback

1. If an imported dashboard overwrote an existing one, restore from the pre-import JSON backup
2. Delete incorrectly imported dashboards via Grafana API: `curl -X DELETE -H "Authorization: Bearer <api-key>" https://<endpoint>/api/dashboards/uid/<uid>`
3. If data source UIDs were changed in the workspace, restore original data source configuration
4. Verify existing dashboards are unaffected by the import operation
5. Document which dashboards were imported and their plugin dependencies

## Output Format

```yaml
root_cause: "import_export_issue — <specific_cause>"
evidence:
  - type: dashboard_json
    content: "<plugin and data source references in JSON>"
  - type: workspace_plugins
    content: "<available plugins in workspace>"
  - type: workspace_data_sources
    content: "<configured data sources in workspace>"
severity: LOW
mitigation:
  immediate: "Update dashboard JSON to match workspace capabilities"
  long_term: "Standardize dashboard templates with portable data source references"
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
