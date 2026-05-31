---
title: "D1 — Dashboard Errors"
description: "Diagnose dashboard rendering errors and blank panels in Amazon Managed Grafana"
status: active
severity: MEDIUM
triggers:
  - "dashboard error"
  - "blank panel"
  - "panel not rendering"
  - "dashboard broken"
  - "visualization error"
owner: devops-agent
objective: "Identify why dashboards display errors or blank panels and restore visualization"
context: "Dashboard errors in AMG can stem from missing plugins, data source misconfiguration, incompatible panel types, query errors, or permission issues. Panels may show 'No data', error messages, or fail to render entirely."
---

## Phase 1 — Triage

MUST:
- Check the dashboard JSON for panel configurations via Grafana API: `curl -H "Authorization: Bearer <api-key>" https://<workspace-endpoint>/api/dashboards/uid/<dashboard-uid>`
- Identify which panels are failing and their data source references
- Verify referenced data sources exist and are configured: `curl -H "Authorization: Bearer <api-key>" https://<workspace-endpoint>/api/datasources`
- Check if panels use plugins that are available in the workspace: `aws grafana describe-workspace-configuration --workspace-id <workspace-id>`

SHOULD:
- Test data source connectivity from the Grafana UI (Data Sources > Test)
- Check for query errors in the panel inspect view (Panel > Inspect > Query)
- Verify the user has permission to view the dashboard and query the data sources

MAY:
- Check browser console for JavaScript errors
- Compare the dashboard JSON with a known-working version

## Phase 2 — Remediate

MUST:
- Fix data source references if they point to non-existent data sources
- Enable required plugins if panels depend on unavailable plugins
- Correct query syntax errors in failing panels

SHOULD:
- Replace unsupported panel types with available alternatives
- Update variable queries if they reference incorrect data sources
- Re-save the dashboard after fixing panel configurations

MAY:
- Export the dashboard JSON, fix issues, and re-import
- Create a new dashboard from scratch if the existing one is heavily corrupted

## Common Issues

- symptoms: "Panel shows 'Plugin not found' error"
  diagnosis: "Dashboard uses a Grafana plugin not available in the AMG plugin catalog."
  resolution: "Replace the panel with an available panel type, or enable the plugin if it's in the AMG catalog."

- symptoms: "All panels show 'No data'"
  diagnosis: "Data source is misconfigured, IAM role lacks permissions, or the time range has no data."
  resolution: "Test the data source connection. Check IAM role permissions. Adjust the time range."

- symptoms: "Dashboard loads but some panels show error"
  diagnosis: "Individual panel queries have syntax errors or reference deleted metrics/indices."
  resolution: "Edit the failing panels and fix the query syntax or metric references."

## Safety Ratings

```
safety_ratings:
  - "check dashboard JSON via API: GREEN — read-only dashboard inspection"
  - "check data sources via API: GREEN — read-only enumeration"
  - "fix data source references in panels: YELLOW — modifies dashboard configuration"
  - "re-import dashboard from JSON: YELLOW — overwrites existing dashboard"
```

## Escalation Conditions

- Production monitoring dashboards are broken and teams cannot view critical metrics
- Dashboard corruption requires re-creation from scratch
- Plugin dependencies cannot be resolved within the AMG plugin catalog
- Dashboard errors affect multiple workspaces simultaneously

## Data Sensitivity

- Dashboard JSON: MEDIUM — may contain query patterns revealing infrastructure details
- Data source configuration: MEDIUM — reveals data source endpoints and auth settings
- Plugin configuration: LOW — reveals which plugins are enabled

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER delete a production dashboard without first exporting a JSON backup
- NEVER modify shared dashboards without notifying the team

## Phase 3 — Rollback

1. If a dashboard was modified incorrectly, restore from JSON backup via Grafana API
2. Import the backup: `curl -X POST -H "Authorization: Bearer <api-key>" -H "Content-Type: application/json" -d @dashboard-backup.json https://<endpoint>/api/dashboards/db`
3. If data source references were changed, restore the original UIDs in the dashboard JSON
4. Verify all panels render correctly after rollback
5. Notify the team about the dashboard restoration

## Output Format

```yaml
root_cause: "dashboard_error — <specific_cause>"
evidence:
  - type: dashboard_json
    content: "<panel configuration and data source references>"
  - type: data_sources
    content: "<available data sources and their status>"
  - type: plugins
    content: "<available plugins in workspace>"
severity: MEDIUM
mitigation:
  immediate: "Fix panel data source references, queries, or plugin dependencies"
  long_term: "Validate dashboard dependencies before deployment"
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
