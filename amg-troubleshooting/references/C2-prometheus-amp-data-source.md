---
title: "C2 — Prometheus (AMP) Data Source Issues"
description: "Diagnose Amazon Managed Prometheus data source connectivity and query problems in AMG"
status: active
severity: HIGH
triggers:
  - "Prometheus data source"
  - "AMP data source"
  - "Prometheus query failed"
  - "PromQL error"
  - "Managed Prometheus"
owner: devops-agent
objective: "Identify why Prometheus/AMP data source queries fail in AMG and restore metric access"
context: "Amazon Managed Service for Prometheus (AMP) integrates with AMG as a native data source. It requires the workspace IAM role to have AMP read permissions and correct endpoint configuration. Issues typically involve IAM permissions, endpoint URL format, or PromQL query syntax."
---

## Phase 1 — Triage

MUST:
- Verify PROMETHEUS is enabled as a data source type: `aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.dataSources'`
- Check workspace IAM role for AMP permissions: `aws iam list-attached-role-policies --role-name <workspace-role-name>`
- Verify the role has AMP read permissions (aps:QueryMetrics, aps:GetMetricMetadata, aps:GetSeries, aps:GetLabels)
- List AMP workspaces to verify endpoint: `aws amp list-workspaces`
- Get AMP workspace details: `aws amp describe-workspace --workspace-id <amp-workspace-id>`

SHOULD:
- Verify the Prometheus data source URL format: `https://aps-workspaces.<region>.amazonaws.com/workspaces/<amp-workspace-id>`
- Check that SigV4 authentication is configured on the data source
- Verify the AMP workspace is in ACTIVE status

MAY:
- Test PromQL queries directly against AMP: `aws amp query --workspace-id <amp-workspace-id> --query-string 'up'`
- Check if the AMP workspace has ingested data recently

## Phase 2 — Remediate

MUST:
- Add AMP read permissions to the workspace IAM role (AmazonPrometheusQueryAccess managed policy)
- Configure the data source URL with the correct AMP workspace endpoint
- Enable SigV4 authentication on the Prometheus data source in Grafana

SHOULD:
- Set the default region on the data source to match the AMP workspace region
- Verify data is being ingested into the AMP workspace before querying
- Test with a simple PromQL query like `up` to confirm connectivity

MAY:
- Configure multiple Prometheus data sources for different AMP workspaces
- Set up recording rules in AMP for frequently used queries

## Common Issues

- symptoms: "Prometheus data source test fails with 403 Forbidden"
  diagnosis: "Workspace IAM role lacks AMP query permissions or SigV4 auth is not configured."
  resolution: "Attach AmazonPrometheusQueryAccess policy and enable SigV4 authentication on the data source."

- symptoms: "Data source connects but queries return no data"
  diagnosis: "AMP workspace has no ingested data, or the PromQL query targets non-existent metrics."
  resolution: "Verify data ingestion into AMP. Use the Explore view to browse available metrics."

- symptoms: "Data source URL format error"
  diagnosis: "Prometheus data source URL does not follow the AMP endpoint format."
  resolution: "Use the format: https://aps-workspaces.<region>.amazonaws.com/workspaces/<amp-workspace-id>"

## Safety Ratings

```
safety_ratings:
  - "describe-workspace (dataSources): GREEN — read-only check"
  - "amp describe-workspace: GREEN — read-only AMP status check"
  - "attach AmazonPrometheusQueryAccess: YELLOW — grants AMP read access"
  - "enable SigV4 on data source: YELLOW — changes authentication method"
```

## Escalation Conditions

- Prometheus data source failure affects production monitoring dashboards
- AMP workspace is in a different account requiring cross-account IAM setup
- SigV4 configuration changes affect multiple data sources
- AMP workspace has no ingested data, indicating a collector-side issue

## Data Sensitivity

- `aws amp describe-workspace`: LOW — workspace metadata
- `aws iam list-attached-role-policies`: MEDIUM — reveals IAM permissions
- AMP metrics data: MEDIUM — contains application and infrastructure metrics

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER grant AMP write permissions (RemoteWrite) to the Grafana workspace IAM role
- NEVER configure basic auth for AMP data source (only SigV4 is supported)

## Phase 3 — Rollback

1. If IAM policies were changed incorrectly, detach them: `aws iam detach-role-policy --role-name <role> --policy-arn <policy-arn>`
2. If data source URL was changed, restore the correct AMP endpoint format
3. If SigV4 was disabled, re-enable it on the Prometheus data source
4. Verify Grafana dashboards display AMP metrics correctly after rollback
5. Document IAM and data source configuration changes

## Output Format

```yaml
root_cause: "prometheus_data_source — <specific_cause>"
evidence:
  - type: data_source_config
    content: "<Prometheus data source URL and auth settings>"
  - type: iam_role
    content: "<workspace IAM role AMP permissions>"
  - type: amp_workspace
    content: "<AMP workspace status and endpoint>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions, data source URL, or SigV4 configuration"
  long_term: "Use managed policies and standardize AMP data source configuration"
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
