---
title: "G1 — Grafana Data Source"
description: "Diagnose and resolve Grafana data source configuration for AMP"
status: active
severity: MEDIUM
triggers:
  - "Grafana"
  - "data source"
  - "datasource"
  - "Grafana AMP"
  - "Grafana Prometheus"
  - "dashboard not loading"
owner: devops-agent
objective: "Identify and fix Grafana data source configuration for querying AMP"
context: "Grafana connects to AMP as a Prometheus data source using SigV4 authentication. Both Amazon Managed Grafana (AMG) and self-hosted Grafana support AMP. AMG has built-in SigV4 support. Self-hosted Grafana requires the SigV4 auth proxy or Grafana's built-in SigV4 support (Grafana 7.3.5+)."
---

## Phase 1 — Triage

MUST:
- Verify the Grafana data source URL: should be `https://aps-workspaces.<region>.amazonaws.com/workspaces/<workspace-id>`
- Test the data source connection from Grafana's data source settings page ("Save & Test")
- Check if SigV4 authentication is enabled in the data source configuration
- Verify the IAM role/credentials used by Grafana have query permissions

SHOULD:
- For AMG: verify the workspace IAM role has AMP query permissions
- For self-hosted Grafana: verify SigV4 is enabled (`GF_AUTH_SIGV4_AUTH_ENABLED=true` or `[auth] sigv4_auth_enabled = true`)
- Check Grafana server logs for data source connection errors
- Verify the data source type is set to "Prometheus" (not "CloudWatch" or other)

MAY:
- Test the query endpoint directly with curl to isolate Grafana vs AMP issues
- Check if Grafana version supports SigV4 (7.3.5+ for built-in support)
- Verify network connectivity from Grafana to AMP endpoint

## Phase 2 — Remediate

MUST:
- If URL wrong: update to `https://aps-workspaces.<region>.amazonaws.com/workspaces/<workspace-id>`
- If SigV4 not enabled: enable it in Grafana configuration and data source settings
- If IAM permissions missing: add aps:QueryMetrics, aps:GetSeries, aps:GetLabels, aps:GetMetricMetadata
- If data source type wrong: change to Prometheus type

SHOULD:
- For AMG: use the built-in AMP data source type for simplified setup
- For self-hosted Grafana: set SigV4 auth with correct region and default auth provider
- Configure the data source with appropriate scrape interval and query timeout
- Set up provisioned data sources via YAML for consistency

MAY:
- Configure multiple AMP data sources for different workspaces
- Set up Grafana variables that work with AMP label queries
- Implement dashboard provisioning for consistent deployments

## Common Issues

- symptoms: "Data source test returns 'HTTP Error Forbidden'"
  diagnosis: "SigV4 authentication failure — credentials or permissions issue."
  resolution: "Verify SigV4 is enabled. Check IAM role has AMP query permissions. Verify region matches."

- symptoms: "Data source test returns 'HTTP Error Not Found'"
  diagnosis: "Incorrect URL — wrong workspace ID or missing path."
  resolution: "URL should NOT include /api/v1/query — Grafana appends the API path automatically. Use base URL only."

- symptoms: "Dashboards show 'No data' but data source test passes"
  diagnosis: "Query time range doesn't match data availability, or metric names are wrong."
  resolution: "Verify metrics exist with a simple query like 'up'. Check dashboard time range. Verify metric names."

- symptoms: "SigV4 option not available in Grafana data source settings"
  diagnosis: "Grafana version too old or SigV4 not enabled in server config."
  resolution: "Upgrade Grafana to 7.3.5+. Set sigv4_auth_enabled=true in grafana.ini or GF_AUTH_SIGV4_AUTH_ENABLED=true env var."

## Safety Ratings

```
safety_ratings:
  - "test data source connection: GREEN — read-only connectivity test"
  - "check Grafana server logs: GREEN — read-only log review"
  - "enable SigV4 in Grafana config: YELLOW — changes Grafana server configuration"
  - "configure data source URL: YELLOW — changes data source endpoint"
```

## Escalation Conditions

- Grafana data source failure affects production monitoring dashboards
- SigV4 configuration requires Grafana server restart (self-hosted)
- IAM permission changes needed for the Grafana service role
- Data source connectivity issues affect multiple AMP workspaces

## Data Sensitivity

- Data source configuration: MEDIUM — reveals AMP endpoint and auth settings
- Grafana server logs: MEDIUM — may contain error details and endpoint URLs
- IAM role permissions: MEDIUM — reveals authorization scope

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER include /api/v1/query in the data source URL (Grafana appends it automatically)
- NEVER disable SigV4 as a troubleshooting step

## Phase 3 — Rollback

1. If data source URL was changed, restore the correct AMP endpoint
2. If SigV4 was disabled in Grafana config, re-enable it and restart Grafana
3. If IAM permissions were changed, restore previous policy
4. Verify dashboards display AMP metrics correctly after rollback
5. Test data source connectivity from Grafana settings page

## Output Format

```yaml
root_cause: "Grafana data source — <specific_cause>"
evidence:
  - type: datasource_config
    content: "<data source URL, auth, and test results>"
  - type: grafana_logs
    content: "<relevant Grafana server logs>"
severity: MEDIUM
mitigation:
  immediate: "Fix data source URL, SigV4 configuration, or IAM permissions"
  long_term: "Use provisioned data sources, implement monitoring for data source health"
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
