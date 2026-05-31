---
title: "A2 — AMG Workspace Upgrade Issues"
description: "Diagnose problems with Amazon Managed Grafana workspace version upgrades"
status: active
severity: MEDIUM
triggers:
  - "workspace upgrade failed"
  - "grafana version upgrade"
  - "upgrade stuck"
  - "version update error"
owner: devops-agent
objective: "Identify why a workspace version upgrade fails or causes issues and resolve it"
context: "AMG workspace upgrades move the Grafana version forward. AWS manages the upgrade process but users can request upgrades. Failures can occur due to incompatible configurations, plugin issues, or workspace state problems. Downgrades are not supported."
---

## Phase 1 — Triage

MUST:
- Check current workspace version and status: `aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.{Version:grafanaVersion,Status:status}'`
- Check CloudTrail for upgrade events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateWorkspace --max-results 10`
- Verify workspace health after upgrade: `aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.health'`

SHOULD:
- Check if dashboards and alerts are functioning post-upgrade
- Review workspace configuration for deprecated features: `aws grafana describe-workspace-configuration --workspace-id <workspace-id>`
- Test data source connectivity after upgrade via Grafana API

MAY:
- Check Grafana release notes for breaking changes in the target version
- Review alert rules for compatibility with the new alerting engine

## Phase 2 — Remediate

MUST:
- If upgrade is stuck, check workspace status and wait for completion (upgrades can take several minutes)
- Verify all data sources are still connected after upgrade
- Confirm alert rules are evaluating correctly post-upgrade

SHOULD:
- Re-test dashboards that use advanced features or custom queries
- Update any automation scripts that depend on version-specific API behavior
- Review and update notification policies if alerting behavior changed

MAY:
- Export critical dashboards as JSON backup before requesting upgrades
- Document version-specific configuration changes for team awareness

## Common Issues

- symptoms: "Workspace status stuck in UPDATING for extended period"
  diagnosis: "Upgrade process is still running or encountered an internal error."
  resolution: "Wait up to 30 minutes. If still stuck, contact AWS Support with the workspace ID."

- symptoms: "Dashboards show errors after upgrade"
  diagnosis: "Dashboard JSON uses deprecated panel types or query syntax from older Grafana version."
  resolution: "Update dashboard panels to use current panel types. Re-save dashboards in the new version."

- symptoms: "Alert rules stopped firing after upgrade"
  diagnosis: "Unified alerting changes in the new version may require rule reconfiguration."
  resolution: "Review alert rules in the Alerting section. Update query expressions and conditions as needed."

## Safety Ratings

```
safety_ratings:
  - "describe-workspace (version/status): GREEN — read-only check"
  - "describe-workspace-configuration: GREEN — read-only check"
  - "export dashboards as JSON backup: GREEN — non-destructive data export"
  - "update alert rules post-upgrade: YELLOW — modifies alerting configuration"
```

## Escalation Conditions

- Workspace stuck in UPDATING status for more than 30 minutes
- Dashboards or alerts broken after upgrade with no clear fix
- Upgrade required for security patch but breaks critical monitoring
- Rollback needed but Grafana version downgrades are not supported

## Data Sensitivity

- `aws grafana describe-workspace`: LOW — returns workspace metadata
- Dashboard JSON export: MEDIUM — may contain query patterns revealing infrastructure
- Alert rule configuration: MEDIUM — reveals monitoring thresholds and notification targets

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER upgrade production workspaces without first testing in a non-production workspace
- NEVER modify alert rules during an active incident

## Phase 3 — Rollback

1. Grafana version downgrades are NOT supported — plan upgrades carefully
2. If dashboards broke after upgrade, restore from JSON backup: import via Grafana API
3. If alert rules stopped working, reconfigure them for the new alerting engine
4. If the workspace is stuck in UPDATING, contact AWS Support with the workspace ID
5. Document all post-upgrade issues and fixes for future upgrade planning

## Output Format

```yaml
root_cause: "workspace_upgrade_issue — <specific_cause>"
evidence:
  - type: workspace_version
    content: "<current version and target version>"
  - type: workspace_status
    content: "<status during/after upgrade>"
  - type: cloudtrail
    content: "<upgrade event details>"
severity: MEDIUM
mitigation:
  immediate: "Address version-specific incompatibilities in dashboards and alerts"
  long_term: "Establish pre-upgrade testing process with dashboard and alert validation"
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
