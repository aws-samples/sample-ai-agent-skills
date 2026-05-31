# Amazon Managed Grafana Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Managed Grafana (AMG) problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Managed Grafana when the console alone isn't enough — workspace health analysis, data source connectivity debugging, authentication/SSO issues, dashboard errors, alerting failures, plugin problems, API access, VPC networking, and IAM permission issues.

### Activate When

- Workspace creation or update failures
- Workspace upgrade or version issues
- Workspace configuration problems
- SSO / IAM Identity Center authentication failures
- SAML configuration issues
- User or role mapping not working
- CloudWatch data source errors
- Prometheus (AMP) data source connectivity failures
- OpenSearch data source issues
- Cross-account data source access denied
- Dashboard rendering errors or blank panels
- Query failures or timeouts
- Dashboard import/export problems
- Alert rule not firing or evaluation errors
- Notification channel delivery failures
- Plugin not available or not loading
- Plugin configuration errors
- VPC connectivity issues for workspace
- Private data source unreachable
- API key authentication failures
- Terraform or CloudFormation provisioning errors

---

## Skill Structure

```
amg-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-workspace-creation-failures.md
    ├── A2-workspace-upgrade-issues.md
    ├── A3-workspace-configuration.md
    ├── B1-sso-iam-identity-center.md
    ├── B2-saml-configuration.md
    ├── B3-user-role-mapping.md
    ├── C1-cloudwatch-data-source.md
    ├── C2-prometheus-amp-data-source.md
    ├── C3-opensearch-data-source.md
    ├── C4-cross-account-data-sources.md
    ├── D1-dashboard-errors.md
    ├── D2-query-failures.md
    ├── D3-import-export-issues.md
    ├── E1-alert-rule-failures.md
    ├── E2-notification-channels.md
    ├── F1-plugin-availability.md
    ├── F2-plugin-configuration.md
    ├── G1-vpc-connectivity.md
    ├── G2-private-data-sources.md
    ├── H1-api-key-issues.md
    ├── H2-terraform-cloudformation.md
    ├── Z1-general-troubleshooting.md
    ├── amg-guardrails.md
    └── amg-hallucination-patterns.yaml
```

---

## Runbook Library (26 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Workspace** | A1–A3 | Creation failures, upgrade issues, configuration |
| **B — Authentication** | B1–B3 | SSO/IAM Identity Center issues, SAML configuration, user/role mapping |
| **C — Data Sources** | C1–C4 | CloudWatch, Prometheus (AMP), OpenSearch, cross-account data sources |
| **D — Dashboards** | D1–D3 | Dashboard errors, query failures, import/export issues |
| **E — Alerting** | E1–E2 | Alert rule failures, notification channels |
| **F — Plugins** | F1–F2 | Plugin availability, plugin configuration |
| **G — Networking** | G1–G2 | VPC connectivity, private data sources |
| **H — API & Automation** | H1–H2 | API key issues, Terraform/CloudFormation provisioning |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/amg-guardrails.md` covering managed service boundaries, authentication modes, workspace IAM role scope, data source permissions vs Grafana RBAC, plugin restrictions, version management, VPC requirements, alerting model, API key scope, workspace pricing, service account best practices, and cross-account trust configuration.

---

## Investigation Workflow

1. **Triage** — List workspaces, describe workspace status/health, check CloudWatch metrics
2. **Deep Dive** — Examine data source config, authentication settings, workspace permissions and IAM role
3. **Detailed** — Review CloudTrail events, VPC configuration, security groups, VPC endpoints

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `grafana:*`, `iam:GetRole`, `iam:ListAttachedRolePolicies`, `sso:*`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`, `ec2:DescribeSecurityGroups`, `ec2:DescribeSubnets`, `ec2:DescribeVpcEndpoints`
- Grafana workspace API access (via workspace endpoint URL) for dashboard and data source operations
- IAM Identity Center or SAML IdP configured for authentication

---

## Usage Examples

```
# List all workspaces
aws grafana list-workspaces

# Get workspace details
aws grafana describe-workspace --workspace-id g-1234567890

# Check authentication configuration
aws grafana describe-workspace-authentication --workspace-id g-1234567890

# List workspace permissions
aws grafana list-permissions --workspace-id g-1234567890

# Check workspace IAM role
aws grafana describe-workspace --workspace-id g-1234567890 --query 'workspace.workspaceRoleArn'

# Query Grafana API for data sources
curl -H "Authorization: Bearer <api-key>" https://<workspace-endpoint>/api/datasources
```

---

## License

MIT-0
