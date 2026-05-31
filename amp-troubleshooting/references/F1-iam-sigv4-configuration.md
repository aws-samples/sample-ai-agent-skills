---
title: "F1 — IAM/SigV4 Configuration"
description: "Diagnose and resolve IAM and SigV4 configuration issues for AMP access"
status: active
severity: HIGH
triggers:
  - "IAM"
  - "permission"
  - "access denied"
  - "SigV4"
  - "role"
  - "policy"
  - "IRSA"
  - "pod identity"
owner: devops-agent
objective: "Identify and fix IAM permission and SigV4 signing configuration for AMP"
context: "AMP uses IAM for authorization and SigV4 for request signing. Different AMP operations require different IAM actions: aps:RemoteWrite for ingestion, aps:QueryMetrics for queries, aps:GetSeries/GetLabels for metadata. Collectors need IAM credentials via IRSA, Pod Identity, instance profiles, or environment variables."
---

## Phase 1 — Triage

MUST:
- Identify the IAM principal (role/user) used by the collector or client
- Simulate IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names aps:RemoteWrite aps:QueryMetrics aps:GetSeries aps:GetLabels aps:GetMetricMetadata`
- Check the IAM policy attached to the role: `aws iam list-attached-role-policies --role-name <role>` and `aws iam list-role-policies --role-name <role>`
- Verify the trust policy allows the expected principal to assume the role

SHOULD:
- For EKS IRSA: verify OIDC provider, service account annotation, and trust policy condition
- For EC2: verify instance profile is attached and role has AMP permissions
- Check for permission boundaries that may restrict AMP actions
- Check for SCPs that may deny AMP actions at the organization level

MAY:
- Use CloudTrail to find specific AccessDenied events for AMP
- Check if resource-level permissions are scoped to the correct workspace ARN
- Verify no explicit deny in any policy in the evaluation chain

## Phase 2 — Remediate

MUST:
- Create or update IAM policy with required AMP permissions:
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "aps:RemoteWrite",
          "aps:QueryMetrics",
          "aps:GetSeries",
          "aps:GetLabels",
          "aps:GetMetricMetadata"
        ],
        "Resource": "arn:aws:aps:<region>:<account>:workspace/<workspace-id>"
      }
    ]
  }
  ```
- For IRSA: ensure trust policy includes the OIDC provider and service account condition
- For Pod Identity: configure the EKS Pod Identity association
- Verify SigV4 signing region matches the workspace region

SHOULD:
- Use least-privilege: scope permissions to specific workspace ARN
- Separate read (QueryMetrics) and write (RemoteWrite) roles for different use cases
- Use managed policies where available (AmazonPrometheusRemoteWriteAccess, AmazonPrometheusQueryAccess)

MAY:
- Implement cross-account access using role chaining for centralized AMP workspaces
- Set up IAM Access Analyzer to monitor AMP access patterns
- Create custom managed policies for team-specific AMP access

## Common Issues

- symptoms: "AccessDeniedException: User is not authorized to perform aps:RemoteWrite"
  diagnosis: "IAM policy missing aps:RemoteWrite permission."
  resolution: "Attach AmazonPrometheusRemoteWriteAccess managed policy or add aps:RemoteWrite to custom policy."

- symptoms: "IRSA not working — collector uses node role instead of service account role"
  diagnosis: "Service account not annotated, OIDC provider missing, or trust policy incorrect."
  resolution: "Verify all three IRSA components: OIDC provider, SA annotation, trust policy condition."

- symptoms: "Cross-account access denied"
  diagnosis: "Trust policy doesn't allow the source account's role to assume the target role."
  resolution: "Update trust policy to allow sts:AssumeRole from the source account role. Add sts:AssumeRole permission in source account."

- symptoms: "Permissions work for query but not remote write (or vice versa)"
  diagnosis: "IAM policy only includes some AMP actions, not all needed ones."
  resolution: "Add the missing actions. RemoteWrite and QueryMetrics are separate IAM actions."

## Safety Ratings

```
safety_ratings:
  - "simulate-principal-policy: GREEN — read-only IAM check"
  - "list-attached-role-policies: GREEN — read-only policy enumeration"
  - "add AMP permissions to IAM policy: YELLOW — changes IAM authorization"
  - "configure IRSA trust policy: YELLOW — changes EKS service account trust"
  - "configure cross-account role chaining: RED — establishes cross-account trust"
```

## Escalation Conditions

- IAM permission issues causing data loss in production monitoring
- IRSA configuration requires EKS cluster OIDC provider changes
- Cross-account access requires trust policy changes in other accounts
- SCP or permission boundary blocks AMP actions at the organization level

## Data Sensitivity

- IAM policy simulation: LOW — tests permissions without changes
- IAM policies and trust policies: MEDIUM — reveals authorization rules
- IRSA configuration: MEDIUM — reveals service account trust relationships
- Cross-account role configuration: HIGH — reveals inter-account trust

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER grant aps:* wildcard permissions — scope to specific actions and workspace ARN
- NEVER use static access keys when IAM roles are available

## Phase 3 — Rollback

1. If IAM policies were changed incorrectly, detach new policies and reattach original ones
2. If IRSA trust policy was modified, restore previous OIDC condition
3. If cross-account role was configured incorrectly, remove trust policy entry in target account
4. Verify all collectors and clients can authenticate after rollback
5. Confirm metrics ingestion and query access are functioning

## Output Format

```yaml
root_cause: "IAM/SigV4 configuration — <specific_cause>"
evidence:
  - type: iam_simulation
    content: "<policy simulation results>"
  - type: iam_policy
    content: "<attached policies and trust policy>"
severity: HIGH
mitigation:
  immediate: "Fix IAM policy or trust relationship"
  long_term: "Use least-privilege policies, implement IRSA/Pod Identity, monitor with IAM Access Analyzer"
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
