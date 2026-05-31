---
title: "A3 — AMG Workspace Configuration Issues"
description: "Diagnose Amazon Managed Grafana workspace configuration problems"
status: active
severity: MEDIUM
triggers:
  - "workspace configuration"
  - "workspace settings"
  - "workspace update failed"
  - "configuration error"
owner: devops-agent
objective: "Identify and resolve workspace configuration issues affecting functionality"
context: "AMG workspace configuration includes data source types, notification destinations, authentication settings, VPC configuration, and plugin settings. Misconfigurations can cause data source failures, authentication problems, or feature unavailability."
---

## Phase 1 — Triage

MUST:
- Get full workspace configuration: `aws grafana describe-workspace --workspace-id <workspace-id>`
- Check workspace configuration details: `aws grafana describe-workspace-configuration --workspace-id <workspace-id>`
- Verify enabled data source types: `aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.dataSources'`
- Check notification destinations: `aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.notificationDestinations'`

SHOULD:
- Verify workspace IAM role permissions match enabled data sources
- Check authentication configuration: `aws grafana describe-workspace-authentication --workspace-id <workspace-id>`
- Review recent configuration changes in CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateWorkspace --max-results 10`

MAY:
- Compare configuration with a known-good workspace
- Check workspace tags for organizational context: `aws grafana list-tags-for-resource --resource-arn <workspace-arn>`

## Phase 2 — Remediate

MUST:
- Enable required data source types if missing: `aws grafana update-workspace --workspace-id <workspace-id> --data-sources CLOUDWATCH PROMETHEUS`
- Enable notification destinations if alerting is needed: `aws grafana update-workspace --workspace-id <workspace-id> --notification-destinations SNS`
- Ensure workspace IAM role has permissions for all enabled data source types

SHOULD:
- Update workspace configuration for required plugins: `aws grafana update-workspace-configuration --workspace-id <workspace-id> --configuration <json>`
- Document the workspace configuration for team reference

MAY:
- Use IaC to manage workspace configuration consistently across environments

## Common Issues

- symptoms: "Data source type not available in Grafana UI"
  diagnosis: "The data source type is not enabled in the workspace configuration."
  resolution: "Update workspace to include the data source type: aws grafana update-workspace --data-sources <types>"

- symptoms: "Cannot configure SNS notification channels"
  diagnosis: "SNS is not enabled as a notification destination on the workspace."
  resolution: "Update workspace: aws grafana update-workspace --notification-destinations SNS"

- symptoms: "Workspace update fails with validation error"
  diagnosis: "Invalid combination of settings or missing required fields in the update request."
  resolution: "Review the update request parameters. Ensure all required fields are included and values are valid."

## Safety Ratings

```
safety_ratings:
  - "describe-workspace: GREEN — read-only configuration check"
  - "describe-workspace-configuration: GREEN — read-only check"
  - "update-workspace (enable data sources): YELLOW — changes workspace capabilities"
  - "update-workspace (notification destinations): YELLOW — enables new integration points"
```

## Escalation Conditions

- Workspace update fails with validation errors that cannot be resolved
- Configuration changes require enabling data sources that need cross-account IAM setup
- Workspace configuration drift detected between IaC and actual state
- Multiple configuration changes needed simultaneously may cause workspace instability

## Data Sensitivity

- `aws grafana describe-workspace`: LOW — returns workspace metadata
- `aws grafana describe-workspace-configuration`: MEDIUM — reveals enabled plugins and features
- `aws grafana describe-workspace-authentication`: MEDIUM — reveals auth provider configuration

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER disable data source types that are actively used by dashboards
- NEVER change authentication mode on a production workspace without migration planning

## Phase 3 — Rollback

1. If data source types were changed incorrectly, restore them: `aws grafana update-workspace --workspace-id <id> --data-sources <original-types>`
2. If notification destinations were changed, restore: `aws grafana update-workspace --workspace-id <id> --notification-destinations <original-destinations>`
3. If workspace configuration was updated incorrectly: `aws grafana update-workspace-configuration --workspace-id <id> --configuration <original-config>`
4. Verify all dashboards and alerts function correctly after rollback
5. Sync IaC state with the restored configuration

## Output Format

```yaml
root_cause: "workspace_configuration — <specific_cause>"
evidence:
  - type: workspace_config
    content: "<workspace configuration details>"
  - type: data_sources
    content: "<enabled data source types>"
  - type: notification_destinations
    content: "<enabled notification destinations>"
severity: MEDIUM
mitigation:
  immediate: "Update workspace configuration to enable required features"
  long_term: "Manage workspace configuration via IaC for consistency"
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
