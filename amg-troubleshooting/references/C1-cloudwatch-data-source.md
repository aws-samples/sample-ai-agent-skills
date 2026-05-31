---
title: "C1 — CloudWatch Data Source Issues"
description: "Diagnose CloudWatch data source connectivity and query problems in Amazon Managed Grafana"
status: active
severity: HIGH
triggers:
  - "CloudWatch data source"
  - "CloudWatch query failed"
  - "CloudWatch metrics not showing"
  - "CloudWatch access denied"
owner: devops-agent
objective: "Identify why CloudWatch data source queries fail in AMG and restore metric visibility"
context: "CloudWatch is the most common data source for AMG. It requires the workspace IAM role to have CloudWatch read permissions. Issues typically involve IAM permissions, region configuration, metric namespace availability, or query syntax."
---

## Phase 1 — Triage

MUST:
- Verify CloudWatch is enabled as a data source type: `aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.dataSources'`
- Check workspace IAM role permissions: `aws iam list-attached-role-policies --role-name <workspace-role-name>`
- Verify the role has CloudWatch read permissions (cloudwatch:GetMetricData, cloudwatch:ListMetrics, cloudwatch:GetMetricStatistics, cloudwatch:DescribeAlarms)
- Test CloudWatch data source connectivity via Grafana API: `curl -H "Authorization: Bearer <api-key>" https://<workspace-endpoint>/api/datasources`

SHOULD:
- Check the data source configuration for correct default region
- Verify the target metrics exist in CloudWatch: `aws cloudwatch list-metrics --namespace <namespace> --region <region>`
- Check if cross-account CloudWatch access is needed and configured

MAY:
- Check CloudWatch Logs permissions if using Logs queries (logs:GetQueryResults, logs:StartQuery, logs:StopQuery)
- Verify custom namespace metrics are being published

## Phase 2 — Remediate

MUST:
- Add required CloudWatch permissions to the workspace IAM role
- Ensure the data source region matches where metrics are published
- Verify the CloudWatch data source is properly configured in Grafana

SHOULD:
- Use the AWS managed policy CloudWatchReadOnlyAccess for comprehensive read access
- Configure the default region on the data source to match the primary metrics region
- Test with a simple metric query (e.g., AWS/EC2 CPUUtilization) to confirm connectivity

MAY:
- Configure multiple CloudWatch data sources for different regions
- Set up cross-account role assumption for multi-account monitoring

## Common Issues

- symptoms: "CloudWatch data source test fails with access denied"
  diagnosis: "Workspace IAM role lacks CloudWatch read permissions."
  resolution: "Attach CloudWatchReadOnlyAccess or equivalent policy to the workspace IAM role."

- symptoms: "Metrics show in CloudWatch console but not in Grafana"
  diagnosis: "Data source region does not match the region where metrics are published."
  resolution: "Update the CloudWatch data source default region to match the metrics region."

- symptoms: "CloudWatch Logs queries fail"
  diagnosis: "Workspace IAM role lacks CloudWatch Logs permissions (logs:StartQuery, logs:GetQueryResults)."
  resolution: "Add CloudWatch Logs read permissions to the workspace IAM role."

## Safety Ratings

```
safety_ratings:
  - "describe-workspace (dataSources): GREEN — read-only check"
  - "list-attached-role-policies: GREEN — read-only IAM check"
  - "attach CloudWatchReadOnlyAccess policy: YELLOW — grants read access to all CloudWatch data"
  - "configure cross-account CloudWatch: YELLOW — establishes cross-account trust"
```

## Escalation Conditions

- CloudWatch data source failure affects production monitoring dashboards
- IAM role changes require cross-team approval
- Cross-account CloudWatch access needs trust policy changes in target accounts
- CloudWatch Logs permissions needed but security team restricts log access

## Data Sensitivity

- `aws grafana describe-workspace`: LOW — workspace metadata
- `aws iam list-attached-role-policies`: MEDIUM — reveals IAM permissions
- CloudWatch metrics and logs: HIGH — may contain application performance and error data

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER grant CloudWatch write permissions to the workspace IAM role
- NEVER attach AdministratorAccess to resolve CloudWatch permission issues

## Phase 3 — Rollback

1. If IAM policies were changed incorrectly, detach them: `aws iam detach-role-policy --role-name <role> --policy-arn <policy-arn>`
2. If data source region was changed, restore the original region in Grafana data source settings
3. If cross-account access was configured incorrectly, remove the trust policy in the target account
4. Verify dashboards display data correctly after rollback
5. Document IAM changes for security audit

## Output Format

```yaml
root_cause: "cloudwatch_data_source — <specific_cause>"
evidence:
  - type: data_source_config
    content: "<CloudWatch data source configuration>"
  - type: iam_role
    content: "<workspace IAM role permissions>"
  - type: metrics
    content: "<available metrics in target namespace>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions or data source region configuration"
  long_term: "Use managed policies and document required permissions per data source type"
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
