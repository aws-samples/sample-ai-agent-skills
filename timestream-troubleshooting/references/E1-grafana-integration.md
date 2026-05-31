---
title: "E1 — Grafana Integration Issues"
description: "Diagnose Timestream integration with Grafana"
status: active
severity: MEDIUM
triggers:
  - "Grafana Timestream"
  - "Grafana plugin"
  - "Grafana data source"
  - "Grafana query error"
owner: devops-agent
objective: "Identify and resolve Grafana-Timestream integration issues"
context: "Grafana connects to Timestream via the Amazon Timestream plugin. It requires IAM credentials (access keys or IAM role), correct region configuration, and the Timestream plugin installed. Amazon Managed Grafana has built-in Timestream support."
---

## Phase 1 — Triage

MUST:
- Verify the Timestream plugin is installed in Grafana
- Check the data source configuration (region, authentication)
- Test the data source connection in Grafana
- Verify IAM credentials have timestream:Select and timestream:DescribeEndpoints permissions

SHOULD:
- Check Grafana server logs for Timestream connection errors
- Verify the Grafana server can reach Timestream endpoints (network/firewall)
- Test a simple query in the Grafana query editor

MAY:
- Check if using Amazon Managed Grafana (AMG) with service-managed permissions
- Verify proxy settings if Grafana is behind a proxy

## Phase 2 — Remediate

MUST:
- Install the Amazon Timestream plugin if missing
- Configure the data source with correct region and credentials
- Fix IAM permissions for the Grafana service account

SHOULD:
- Use IAM roles instead of access keys where possible
- For AMG, configure the Timestream data source via the workspace settings
- Test with a simple query before building dashboards

MAY:
- Set up Grafana alerting based on Timestream queries
- Create dashboard templates for common Timestream queries

## Common Issues

- symptoms: "Data source test fails with authentication error"
  diagnosis: "IAM credentials invalid or missing Timestream permissions."
  resolution: "Verify credentials and add timestream:Select, timestream:DescribeEndpoints permissions."

- symptoms: "Plugin not available in Grafana"
  diagnosis: "Timestream plugin not installed."
  resolution: "Install the Amazon Timestream plugin from the Grafana plugin catalog."

- symptoms: "Query works in AWS console but fails in Grafana"
  diagnosis: "Grafana query syntax differs slightly from raw SQL."
  resolution: "Use the Grafana query builder or adjust SQL syntax for the plugin."

## Output Format

```yaml
root_cause: "grafana_integration_issue — <specific_cause>"
evidence:
  - type: plugin_status
    content: "<Timestream plugin installation status>"
  - type: data_source_config
    content: "<data source configuration>"
  - type: iam_permissions
    content: "<IAM permissions for Grafana>"
severity: MEDIUM
mitigation:
  immediate: "Fix Grafana data source configuration"
  long_term: "Use IAM roles and standardize Grafana-Timestream setup"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Time-series data may contain IoT sensor readings and telemetry
- Query results may reveal operational patterns and metrics
- Scheduled query outputs may aggregate sensitive measurements
- Classification: CONFIDENTIAL - operational telemetry and measurement data

## Prohibited Actions

- NEVER delete Timestream tables containing production time-series data
- NEVER reduce memory store retention below the current data window
- NEVER modify magnetic store retention without data lifecycle review
- NEVER cancel scheduled queries that feed production dashboards

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
