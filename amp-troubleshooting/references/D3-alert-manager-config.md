---
title: "D3 — Alert Manager Configuration"
description: "Diagnose and resolve AMP alert manager configuration issues"
status: active
severity: MEDIUM
triggers:
  - "alert manager"
  - "alertmanager"
  - "alert config"
  - "routing"
  - "receiver"
  - "inhibit"
  - "silence"
owner: devops-agent
objective: "Identify and fix alert manager configuration issues in AMP"
context: "AMP alert manager configuration defines routing trees, receivers (SNS only), grouping, inhibition rules, and silences. Configuration is uploaded as YAML via AWS API. AMP supports a subset of Alertmanager configuration — only SNS receivers are supported."
---

## Phase 1 — Triage

MUST:
- Get current alert manager definition: `aws amp describe-alert-manager-definition --workspace-id <workspace-id>`
- Check definition status: look for `status.statusCode` (ACTIVE, CREATION_FAILED, UPDATE_FAILED)
- Decode the definition data (base64): the data field contains the YAML configuration
- Verify the YAML structure matches AMP's supported format

SHOULD:
- Check if routes match the expected alert labels
- Verify all receivers referenced in routes are defined
- Check group_by, group_wait, group_interval, repeat_interval settings
- Verify inhibition rules are correctly configured

MAY:
- Compare configuration against a known-good backup
- Check CloudTrail for recent PutAlertManagerDefinition or DeleteAlertManagerDefinition events
- Review if silences are suppressing expected alerts

## Phase 2 — Remediate

MUST:
- If definition missing: create it:
  ```
  aws amp create-alert-manager-definition --workspace-id <workspace-id> --data file://alertmanager.yaml
  ```
- If definition invalid: fix YAML and update:
  ```
  aws amp put-alert-manager-definition --workspace-id <workspace-id> --data file://alertmanager.yaml
  ```
- Ensure all receivers use sns_configs only
- Ensure route tree has a default receiver

SHOULD:
- Structure routing tree to match alert severity and team ownership
- Configure appropriate grouping to reduce notification noise
- Set reasonable timing: group_wait=30s, group_interval=5m, repeat_interval=4h
- Back up alert manager configuration in version control

MAY:
- Implement multiple SNS topics for different alert severities
- Configure inhibition rules to suppress dependent alerts
- Use templates for consistent alert message formatting

## Common Issues

- symptoms: "PutAlertManagerDefinition returns ValidationException"
  diagnosis: "YAML syntax error or unsupported configuration option."
  resolution: "Check YAML syntax. Ensure only sns_configs receivers are used. Remove unsupported fields."

- symptoms: "Alerts going to wrong receiver"
  diagnosis: "Route matching is incorrect — labels don't match route matchers."
  resolution: "Review route tree. Routes are matched top-down, first match wins. Check label matchers in each route."

- symptoms: "Too many duplicate notifications"
  diagnosis: "group_by configuration is too granular or repeat_interval is too short."
  resolution: "Adjust group_by to aggregate related alerts. Increase repeat_interval to reduce duplicates."

- symptoms: "Some alerts are silenced unexpectedly"
  diagnosis: "Inhibition rules or silences are suppressing the alerts."
  resolution: "Review inhibition rules in the config. Check active silences via the alertmanager API."

## Safety Ratings

```
safety_ratings:
  - "describe-alert-manager-definition: GREEN — read-only config retrieval"
  - "decode base64 definition: GREEN — local data processing"
  - "put-alert-manager-definition: YELLOW — updates alerting configuration"
  - "create-alert-manager-definition: YELLOW — creates alerting configuration"
```

## Escalation Conditions

- Alert manager configuration changes affect all alerting for the workspace
- Routing tree changes may cause alerts to go to wrong teams
- Inhibition rules may suppress critical alerts
- Configuration upload fails repeatedly with validation errors

## Data Sensitivity

- Alert manager YAML: MEDIUM — reveals alerting logic, routing, and SNS topic ARNs
- Route matchers: MEDIUM — reveals alert classification and team ownership
- Inhibition rules: MEDIUM — reveals alert dependency relationships

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER remove the default receiver from the route tree
- NEVER use non-SNS receiver types (slack_configs, email_configs, etc.)

## Phase 3 — Rollback

1. Restore previous alert manager definition from version control: `aws amp put-alert-manager-definition --workspace-id <id> --data file://previous-alertmanager.yaml`
2. If the definition was deleted, recreate it: `aws amp create-alert-manager-definition --workspace-id <id> --data file://alertmanager.yaml`
3. Verify routing tree sends alerts to correct receivers after rollback
4. Confirm inhibition rules are not suppressing critical alerts
5. Test end-to-end alert notification delivery

## Output Format

```yaml
root_cause: "Alert manager configuration — <specific_cause>"
evidence:
  - type: alertmanager_definition
    content: "<current configuration and status>"
  - type: routing_analysis
    content: "<route matching analysis>"
severity: MEDIUM
mitigation:
  immediate: "Fix the alert manager configuration and re-upload"
  long_term: "Version control configurations, implement CI/CD validation"
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
