---
title: "F2 — Plugin Configuration Issues"
description: "Diagnose plugin configuration problems in Amazon Managed Grafana"
status: active
severity: MEDIUM
triggers:
  - "plugin configuration"
  - "plugin not working"
  - "plugin error"
  - "plugin settings"
  - "plugin enable"
owner: devops-agent
objective: "Identify why an available plugin is not functioning correctly and fix its configuration"
context: "Even when a plugin is available in the AMG catalog, it may not work correctly due to configuration issues, missing permissions, or incompatible settings. Plugin configuration in AMG is managed through the workspace configuration API and the Grafana UI."
---

## Phase 1 — Triage

MUST:
- Check if the plugin is enabled on the workspace: `aws grafana describe-workspace-configuration --workspace-id <workspace-id>`
- Verify the plugin appears in the Grafana UI Plugins section
- Check plugin-specific configuration in the Grafana UI
- Review error messages from panels or data sources using the plugin

SHOULD:
- Check if the plugin requires additional IAM permissions on the workspace role
- Verify the plugin version is compatible with the workspace Grafana version
- Test the plugin with default settings to isolate configuration issues

MAY:
- Check Grafana API for plugin settings: `curl -H "Authorization: Bearer <api-key>" https://<workspace-endpoint>/api/plugins/<plugin-id>/settings`
- Review plugin documentation for AMG-specific configuration requirements

## Phase 2 — Remediate

MUST:
- Enable the plugin through workspace configuration if disabled
- Configure required plugin settings through the Grafana UI or API
- Add any required IAM permissions to the workspace role

SHOULD:
- Follow the plugin's AMG-specific documentation for configuration
- Test the plugin after configuration changes
- Document plugin configuration for team reference

MAY:
- Use the Grafana provisioning API to manage plugin configuration as code
- Set up monitoring for plugin health

## Common Issues

- symptoms: "Plugin is installed but panels show 'Loading' indefinitely"
  diagnosis: "Plugin is enabled but not properly configured, or it requires additional permissions."
  resolution: "Check plugin settings in the Grafana UI. Verify IAM permissions if the plugin accesses AWS services."

- symptoms: "Data source plugin connects but returns unexpected results"
  diagnosis: "Plugin configuration parameters (endpoint, authentication, query settings) are incorrect."
  resolution: "Review and correct the plugin configuration. Test with the plugin's default settings first."

- symptoms: "Plugin was working but stopped after workspace update"
  diagnosis: "Workspace upgrade changed the Grafana version, and the plugin has compatibility issues."
  resolution: "Check plugin compatibility with the new Grafana version. Update plugin configuration if needed."

## Safety Ratings

```
safety_ratings:
  - "describe-workspace-configuration: GREEN — read-only plugin status check"
  - "check plugin settings via API: GREEN — read-only configuration check"
  - "enable plugin via workspace configuration: YELLOW — changes workspace capabilities"
  - "configure plugin settings: YELLOW — changes plugin behavior"
```

## Escalation Conditions

- Plugin stopped working after a workspace upgrade
- Plugin requires IAM permissions that need cross-team approval
- Plugin configuration changes affect all users of the workspace
- Plugin health issues affect production dashboards or data sources

## Data Sensitivity

- Plugin configuration: MEDIUM — may contain endpoint URLs and auth settings
- Plugin error messages: LOW — general error information
- Workspace configuration: MEDIUM — reveals enabled features and plugins

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER modify plugin configuration on production workspaces without testing
- NEVER disable plugins that are actively used by dashboards or alerts

## Phase 3 — Rollback

1. If plugin configuration was changed incorrectly, restore previous settings via Grafana API
2. If a plugin was enabled and caused issues, disable it via workspace configuration
3. If IAM permissions were added for a plugin, detach them if the plugin is being rolled back
4. Verify all dashboards and data sources using the plugin function correctly after rollback
5. Document plugin configuration changes for future reference

## Output Format

```yaml
root_cause: "plugin_configuration — <specific_cause>"
evidence:
  - type: plugin_status
    content: "<plugin enabled/disabled state>"
  - type: plugin_config
    content: "<plugin configuration settings>"
  - type: error_message
    content: "<specific error from plugin>"
severity: MEDIUM
mitigation:
  immediate: "Fix plugin configuration or enable the plugin"
  long_term: "Document plugin configurations and test after workspace upgrades"
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
