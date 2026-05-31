---
title: "C4 — Cross-Account Data Source Issues"
description: "Diagnose cross-account data source access problems in Amazon Managed Grafana"
status: active
severity: HIGH
triggers:
  - "cross-account"
  - "assume role"
  - "other account"
  - "multi-account"
  - "cross-account data source"
owner: devops-agent
objective: "Identify why cross-account data source access fails in AMG and establish correct trust configuration"
context: "AMG can query AWS data sources (CloudWatch, AMP, OpenSearch) in other AWS accounts by assuming a role in the target account. This requires the workspace IAM role to have sts:AssumeRole permission and the target account to have a trust policy allowing the workspace role. Both sides must be configured correctly."
---

## Phase 1 — Triage

MUST:
- Check workspace IAM role for sts:AssumeRole permission: `aws iam list-attached-role-policies --role-name <workspace-role-name>`
- Get the workspace IAM role ARN: `aws grafana describe-workspace --workspace-id <workspace-id> --query 'workspace.workspaceRoleArn'`
- Verify the target account role exists and check its trust policy: `aws iam get-role --role-name <target-role-name>` (run in target account)
- Check the trust policy allows the workspace role ARN as a principal

SHOULD:
- Verify the target role has the required service permissions (CloudWatch, AMP, etc.)
- Check if there are SCP restrictions in either account blocking cross-account access
- Review the data source configuration in Grafana for the assume role ARN

MAY:
- Test role assumption manually: `aws sts assume-role --role-arn <target-role-arn> --role-session-name test`
- Check CloudTrail in both accounts for AssumeRole events

## Phase 2 — Remediate

MUST:
- Add sts:AssumeRole permission to the workspace IAM role for the target role ARN
- Configure the target account role trust policy to allow the workspace role ARN
- Set the assume role ARN in the Grafana data source configuration

SHOULD:
- Use external ID in the trust policy for additional security
- Limit the workspace role's AssumeRole permission to specific target role ARNs
- Verify the target role has minimum required permissions for the data source type

MAY:
- Use AWS Organizations for centralized cross-account access management
- Set up dedicated monitoring roles in each target account with consistent naming

## Common Issues

- symptoms: "Cross-account data source returns AccessDenied"
  diagnosis: "Workspace IAM role lacks sts:AssumeRole permission for the target role ARN."
  resolution: "Add an IAM policy to the workspace role allowing sts:AssumeRole on the target role ARN."

- symptoms: "AssumeRole fails with 'is not authorized to perform sts:AssumeRole'"
  diagnosis: "Target account role trust policy does not include the workspace role ARN as a trusted principal."
  resolution: "Update the target role trust policy to allow the workspace role ARN."

- symptoms: "Cross-account access works for CloudWatch but not AMP"
  diagnosis: "Target account role has CloudWatch permissions but lacks AMP query permissions."
  resolution: "Add the required AMP permissions (aps:QueryMetrics, etc.) to the target account role."

## Safety Ratings

```
safety_ratings:
  - "describe-workspace (workspaceRoleArn): GREEN — read-only check"
  - "iam get-role (target account): GREEN — read-only trust policy check"
  - "add sts:AssumeRole to workspace role: YELLOW — enables cross-account access"
  - "update target role trust policy: RED — changes who can access the target account"
```

## Escalation Conditions

- Cross-account access requires trust policy changes in accounts managed by other teams
- SCP restrictions in either account block cross-account access
- Target account contains sensitive production data
- Multiple target accounts need simultaneous trust policy updates

## Data Sensitivity

- `aws grafana describe-workspace (workspaceRoleArn)`: LOW — workspace metadata
- Target account role trust policy: HIGH — reveals cross-account trust relationships
- Cross-account data access: HIGH — enables access to monitoring data in other accounts

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER configure cross-account access with overly permissive trust policies (e.g., allowing any principal)
- NEVER use external ID values that are easily guessable

## Phase 3 — Rollback

1. If cross-account access was configured incorrectly, remove the trust policy entry in the target account
2. Remove sts:AssumeRole permission from the workspace role for the target role ARN
3. Remove the assume role ARN from the Grafana data source configuration
4. Verify the workspace can no longer access the target account
5. Document the rollback and notify both account owners

## Output Format

```yaml
root_cause: "cross_account_data_source — <specific_cause>"
evidence:
  - type: workspace_role
    content: "<workspace IAM role ARN and AssumeRole permissions>"
  - type: target_role
    content: "<target account role trust policy and permissions>"
  - type: data_source_config
    content: "<assume role ARN in data source configuration>"
severity: HIGH
mitigation:
  immediate: "Fix IAM trust configuration on both sides"
  long_term: "Standardize cross-account monitoring roles with IaC"
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
