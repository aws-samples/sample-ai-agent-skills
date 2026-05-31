# Amazon Managed Grafana Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Managed Grafana issue.

## Guardrail 1: AMG Is a Managed Service — No Server-Level Access
Amazon Managed Grafana is a fully managed service. You cannot SSH into the underlying instance, modify grafana.ini, install binaries, or access the file system. All configuration is through the AWS API, AWS Console, or the Grafana web UI. Never suggest server-level troubleshooting steps like checking Grafana logs on disk or restarting the Grafana process.

## Guardrail 2: Authentication Is SSO or SAML Only — No Local Users
AMG does not support local Grafana username/password accounts. Authentication must use IAM Identity Center (AWS SSO) or SAML 2.0 with an external identity provider. You cannot create users in the Grafana admin UI. A workspace can use only ONE authentication mode — SSO or SAML, not both simultaneously.

## Guardrail 3: Workspace IAM Role Controls AWS Data Source Access
The IAM role attached to the workspace determines which AWS services the workspace can query. If CloudWatch, AMP, or OpenSearch queries fail with access denied, the issue is almost always the workspace IAM role missing the required service permissions. Check the role's attached policies before investigating data source configuration.

## Guardrail 4: Data Source Permissions ≠ Grafana RBAC
A user with Grafana Admin role can still fail to query a data source if the workspace IAM role lacks permissions to the underlying AWS service. Grafana RBAC controls what users can do in the Grafana UI (create dashboards, manage alerts). AWS IAM controls what data the workspace can access. These are independent permission layers.

## Guardrail 5: Plugins Are Limited to the AWS-Approved Catalog
AMG only supports plugins from the AWS-managed plugin catalog. You cannot install community plugins via grafana-cli, manual upload, or any other method. If a dashboard requires a plugin not in the catalog, those panels will not render. Always verify plugin availability before recommending a plugin-based solution.

## Guardrail 6: Grafana Version Is AWS-Managed and Auto-Upgraded
You cannot pin, manually upgrade, or downgrade the Grafana version on an AMG workspace. AWS manages the version lifecycle and may auto-upgrade workspaces. Workspace version can be checked via describe-workspace. Do not suggest version-specific workarounds that require a different Grafana version than what is deployed.

## Guardrail 7: VPC Configuration Is Required for Private Data Sources
If data sources (self-managed Prometheus, Elasticsearch, databases) are in a VPC and not publicly accessible, the AMG workspace MUST have VPC configuration with appropriate subnets and security groups. Without VPC configuration, the workspace cannot reach private resources. VPC configuration is set at workspace creation or update — it requires workspace modification.

## Guardrail 8: Alerting Uses Unified Alerting — Not Legacy Dashboard Alerts
AMG uses Grafana unified alerting (introduced in Grafana 8+). Legacy dashboard-based alerts are not supported. Alert rules are defined independently from dashboards in the Alerting section. Alert rules use multi-dimensional queries and support multiple notification policies. Do not reference legacy alerting concepts like dashboard alert tabs.

## Guardrail 9: API Keys and Service Accounts Are Workspace-Scoped
API keys and service accounts are scoped to a single workspace. They cannot access other workspaces. Service accounts are the recommended replacement for legacy API keys. API keys have an expiration date and must be rotated. Service accounts support token management and Grafana RBAC role assignment.

## Guardrail 10: Workspace Pricing Is Per Active User
AMG pricing is based on active users. Editors and admins are billed at a higher rate than viewers. An active user is anyone who logs in during the billing period. Monitor user counts and role assignments to manage costs. Assigning unnecessary Editor/Admin roles increases costs.

## Guardrail 11: Service Accounts Replace Legacy API Keys
Legacy Grafana API keys are deprecated in favor of service accounts. Service accounts provide better security with token rotation, RBAC role assignment, and audit logging. When troubleshooting API access, check whether the workspace uses legacy API keys or service accounts, and recommend migration to service accounts.

## Guardrail 12: Cross-Account Data Sources Require IAM Trust Configuration
Accessing AWS data sources (CloudWatch, AMP, OpenSearch) in other AWS accounts requires cross-account IAM role assumption. The workspace IAM role must have sts:AssumeRole permission, and the target account must have a trust policy allowing the workspace role. Both sides of the trust must be configured correctly.
