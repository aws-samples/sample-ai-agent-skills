---
title: "H1 — Cross-Account Role Setup"
description: "Setting up cross-account access properly with roles"
status: active
severity: HIGH
triggers:
  - "cross-account"
  - "cross account access"
  - "access another account"
  - "multi-account"
  - "cross-account role setup"
owner: devops-agent
objective: "Properly configure cross-account role access with security best practices"
context: "Cross-account access via roles requires: (1) target account creates a role with trust policy allowing the source account, (2) source account grants sts:AssumeRole to its principals. This is the recommended pattern for cross-account access. Resource-based policies are an alternative for specific services."
---

## Phase 1 — Triage

MUST:
- Identify source and target accounts and the required access pattern
- In target account — check if the cross-account role exists: `aws iam get-role --role-name <role>`
- Check the trust policy allows the source account: look for source account ID in Principal
- In source account — check if the principal has sts:AssumeRole: `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names sts:AssumeRole --resource-arns <target-role-arn>`

SHOULD:
- Verify both accounts are in the same Organization (simplifies trust with aws:PrincipalOrgID)
- Check for SCPs in both accounts that might block AssumeRole
- Test the assumption: `aws sts assume-role --role-arn <target-role-arn> --role-session-name test`

MAY:
- Check if AWS RAM is more appropriate for the specific resource type
- Review if AWS SSO/Identity Center can simplify the access pattern

## Phase 2 — Remediate

MUST:
- In target account — create role with trust policy:
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::<source-account>:root"},
      "Action": "sts:AssumeRole",
      "Condition": {"StringEquals": {"sts:ExternalId": "<external-id>"}}
    }]
  }
  ```
- Attach permissions policy to the role for the required access
- In source account — grant sts:AssumeRole to the principal:
  ```json
  {
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::<target-account>:role/<role-name>"
  }
  ```

SHOULD:
- Use specific principal ARNs instead of account root in trust policy
- Add ExternalId for third-party access (confused deputy prevention)
- Add aws:PrincipalOrgID condition if both accounts are in the same Organization
- Apply least-privilege to the role's permissions policy

MAY:
- Set up AWS CloudTrail in both accounts to monitor cross-account activity
- Use AWS Config to monitor cross-account role configurations

## Common Issues

- symptoms: "Cross-account role works for admin but not for specific user"
  diagnosis: "The specific user lacks sts:AssumeRole permission in the source account."
  resolution: "Add sts:AssumeRole for the target role ARN to the user's identity policy."

- symptoms: "Cross-account access works but permissions are too broad"
  diagnosis: "The target role has overly permissive policies."
  resolution: "Apply least-privilege to the target role. Use session policies for further restriction."

- symptoms: "Cannot assume role — 'Access Denied' with no details"
  diagnosis: "Check both sides: trust policy AND source identity policy. Also check SCPs."
  resolution: "Verify trust policy Principal, source sts:AssumeRole permission, and SCPs in both accounts."

## Output Format

```yaml
root_cause: "cross_account_setup — <specific_cause>"
evidence:
  - type: target_role
    content: "<role ARN, trust policy, permissions>"
  - type: source_principal
    content: "<principal ARN, sts:AssumeRole permission>"
  - type: connectivity_test
    content: "<assume-role result>"
severity: HIGH
mitigation:
  immediate: "Configure trust policy and source permissions"
  long_term: "Use Organizations, SCPs, and ExternalId for governance"
```


## Safety Ratings

safety_ratings:
  - "Cross-account role setup (trust policy and IAM policy changes across account boundaries): RED — Cross-account access changes affect security boundaries between AWS accounts and can expose resources to external principals"

## Escalation Conditions

escalation_conditions:
  - "ALWAYS escalate to a human for any cross-account role setup or modification"
  - "Escalate if the target role grants access to sensitive resources (IAM, KMS, S3 data, databases)"
  - "Escalate if the trust policy allows an account outside the organization"

## Rollback

If remediation makes things worse:
1. Document the change that was made
2. Revert the trust policy in the target account using `aws iam update-assume-role-policy --role-name <role> --policy-document file://original-trust-policy.json`, and detach or revert the source account's IAM policy
3. Verify access is restored to pre-change state

## Data Sensitivity

- `aws iam get-role` returns trust policy documents revealing all cross-account trust relationships — classification: HIGH
- `aws iam simulate-principal-policy` for sts:AssumeRole reveals cross-account access capabilities — classification: HIGH
- `aws cloudtrail lookup-events` for AssumeRole events reveals cross-account access patterns and source IPs — classification: HIGH

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting AdministratorAccess to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- Always identify the SPECIFIC denied action from CloudTrail before suggesting a fix
- Always scope fixes to the minimum required permissions (least privilege)

## Cleanup

- No temporary resources are created during cross-account role setup

## Data Sensitivity

data_sensitivity:
  - command: "get-account-authorization-details"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-policy-version"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-role"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest Action: * or Resource: * in any policy"
  - "NEVER suggest removing permission boundaries"
  - "NEVER suggest AdministratorAccess for troubleshooting"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
