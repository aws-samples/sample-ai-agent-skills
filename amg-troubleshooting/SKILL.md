---
name: amg-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Managed Grafana (AMG)
  problems by analyzing workspace configurations, data source connectivity,
  dashboard errors, authentication/SSO settings, alerting rules, plugin issues,
  API access, VPC connectivity, and IAM permissions using structured runbooks.
  Activate when: workspace creation or upgrade failures, SSO/SAML authentication
  issues, user/role mapping problems, CloudWatch or Prometheus data source errors,
  OpenSearch connectivity failures, cross-account data source access, dashboard
  rendering errors, query failures, dashboard import/export issues, alert rule
  failures, notification channel problems, plugin availability or configuration
  issues, VPC connectivity problems, private data source access, API key issues,
  Terraform/CloudFormation provisioning errors, or the user says something is
  wrong with Managed Grafana without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Grafana, IAM, SSO, CloudWatch, AMP,
  OpenSearch, VPC, CloudTrail, and optionally SNS and KMS permissions.
  Grafana workspace API access for dashboard and data source operations.
---

# Amazon Managed Grafana Diagnostics

## When to use

Any Amazon Managed Grafana investigation where the console alone is insufficient — workspace health analysis, data source connectivity debugging, authentication/SSO troubleshooting, dashboard errors, alerting failures, plugin issues, API access problems, VPC networking, or IAM permission issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws grafana list-workspaces --query 'workspaces[*].{Id:id,Name:name,Status:status,Endpoint:endpoint}'
version: "1.0.0"
last_updated: "2025-04-12"
aws grafana describe-workspace --workspace-id <workspace-id>
aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.status'
aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.health'
aws cloudwatch get-metric-statistics --namespace AWS/Grafana --metric-name ActiveUserCount --dimensions Name=WorkspaceId,Value=<workspace-id> --start-time <start> --end-time <end> --period 3600 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/Grafana --metric-name RequestCount --dimensions Name=WorkspaceId,Value=<workspace-id> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

### Step 2 — Domain deep dive

```
aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.dataSources'
aws grafana describe-workspace-authentication --workspace-id <workspace-id>
aws grafana list-permissions --workspace-id <workspace-id>
aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.workspaceRoleArn'
aws iam get-role --role-name <workspace-role-name>
aws iam list-attached-role-policies --role-name <workspace-role-name>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=grafana.amazonaws.com --max-results 20
aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.vpcConfiguration'
aws ec2 describe-security-groups --group-ids <sg-id>
aws ec2 describe-subnets --subnet-ids <subnet-ids>
aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=<vpc-id>
```

Read `references/amg-guardrails.md` before concluding on any Amazon Managed Grafana issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `grafana list-workspaces` | List all workspaces and their status |
| `grafana describe-workspace` | Full workspace details, health, config |
| `grafana describe-workspace-authentication` | SSO/SAML authentication configuration |
| `grafana list-permissions` | Workspace user/group permissions |
| `grafana list-workspace-service-accounts` | Service accounts for API access |
| `grafana describe-workspace-configuration` | Workspace plugin and feature config |
| `iam get-role` | Workspace IAM role details |
| `iam list-attached-role-policies` | Policies on workspace role |
| `cloudtrail lookup-events` | Recent Grafana API events |
| `ec2 describe-security-groups` | VPC security group rules |
| `ec2 describe-vpc-endpoints` | VPC endpoints for private data sources |
| `Grafana HTTP API` | Dashboard, data source, alert CRUD via workspace endpoint |

## Gotchas: Amazon Managed Grafana

- AMG is managed Grafana — not self-hosted. You cannot SSH into the instance, modify Grafana configuration files (grafana.ini), or install arbitrary binaries. All configuration is through the AWS API or the Grafana web UI.
- Authentication requires IAM Identity Center (SSO) or SAML — no local Grafana users. You cannot create username/password accounts in the Grafana UI. All user management flows through your identity provider.
- Workspace IAM role controls which AWS data sources can be queried. The role attached to the workspace must have permissions to the target AWS services (CloudWatch, AMP, OpenSearch, etc.). Data source failures are often IAM role permission issues.
- Data source permissions are separate from Grafana RBAC. A user may have Grafana Admin role but still fail to query a data source if the workspace IAM role lacks the necessary AWS service permissions.
- Plugins are limited to the AWS-approved list — no arbitrary community plugins. You cannot install plugins from the Grafana community registry. Only plugins available in the AMG plugin catalog can be enabled.
- Grafana version is managed by AWS and auto-upgraded. You cannot pin a specific Grafana version or skip upgrades. AWS manages the upgrade schedule and may auto-upgrade workspaces.
- Workspace has two authentication modes: SSO and SAML — you cannot use both simultaneously. Choose one authentication provider per workspace. Switching requires reconfiguration.
- API keys have workspace-level scope. API keys (and service accounts) are scoped to a single workspace. They cannot access other workspaces. Service accounts are the recommended replacement for legacy API keys.
- VPC connectivity is required for private data sources. If your data sources (e.g., self-managed Prometheus, OpenSearch in a VPC) are not publicly accessible, you must configure VPC connectivity on the workspace with appropriate subnets and security groups.
- Alerting uses Grafana-native alerting (not legacy alerting). AMG uses the unified alerting system introduced in Grafana 8+. Legacy dashboard alerts are not supported. Alert rules are defined separately from dashboards.
- Dashboard JSON import/export works but plugin dependencies must be available. If a dashboard JSON references a plugin not available in AMG, panels using that plugin will fail to render after import.
- Workspace pricing is per active user/editor. Editors and admins are billed at a higher rate than viewers. Monitor active user counts to manage costs. Viewers are free in some pricing tiers.

## Anti-hallucination rules

1. Always cite specific workspace configurations, IAM policies, or API responses as evidence. Never guess at workspace settings.
2. AMG does NOT support local Grafana users. Never suggest creating username/password accounts in the Grafana UI.
3. Plugins must be from the AWS-approved catalog. Never recommend installing community plugins via grafana-cli or manual methods.
4. Grafana version is AWS-managed. Never suggest manually upgrading or downgrading the Grafana version.
5. VPC configuration is required for private data sources. Never assume a workspace can reach private resources without VPC configuration.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 26 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Workspace | A1-A3 | Creation failures, upgrade issues, configuration |
| B — Authentication | B1-B3 | SSO/IAM Identity Center issues, SAML configuration, user/role mapping |
| C — Data Sources | C1-C4 | CloudWatch, Prometheus (AMP), OpenSearch, cross-account data sources |
| D — Dashboards | D1-D3 | Dashboard errors, query failures, import/export issues |
| E — Alerting | E1-E2 | Alert rule failures, notification channels |
| F — Plugins | F1-F2 | Plugin availability, plugin configuration |
| G — Networking | G1-G2 | VPC connectivity, private data sources |
| H — API & Automation | H1-H2 | API key issues, Terraform/CloudFormation provisioning |
| Z — Catch-All | Z1 | General troubleshooting |
