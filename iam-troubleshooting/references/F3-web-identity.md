---
title: "F3 — Web Identity Federation"
description: "Diagnose web identity federation issues with Cognito and social identity providers"
status: active
severity: MEDIUM
triggers:
  - "web identity"
  - "Cognito"
  - "social login"
  - "Google login AWS"
  - "Facebook login AWS"
owner: devops-agent
objective: "Resolve web identity federation issues with Cognito identity pools and social providers"
context: "Web identity federation allows users to sign in with social IdPs (Google, Facebook, Apple) or Cognito User Pools and receive temporary AWS credentials via Cognito Identity Pools (federated identities). Issues typically involve pool configuration, role mapping, or token validation."
---

## Phase 1 — Triage

MUST:
- Identify the Cognito Identity Pool: `aws cognito-identity list-identity-pools --max-results 20`
- Get pool configuration: `aws cognito-identity describe-identity-pool --identity-pool-id <pool-id>`
- Check role mappings: `aws cognito-identity get-identity-pool-roles --identity-pool-id <pool-id>`
- Verify the authenticated and unauthenticated roles exist and have correct trust policies

SHOULD:
- Check the trust policy on the Cognito roles for `cognito-identity.amazonaws.com` principal
- Verify the identity pool allows the specific provider (Google, Facebook, Cognito User Pool)
- Check CloudTrail for GetCredentialsForIdentity or GetId failures

MAY:
- Verify the social provider's app configuration (client ID, secret)
- Check Cognito User Pool configuration if using as an IdP
- Test token exchange manually with the Cognito API

## Phase 2 — Remediate

MUST:
- Ensure the identity pool role trust policy includes:
  ```json
  {
    "Effect": "Allow",
    "Principal": {"Federated": "cognito-identity.amazonaws.com"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {"cognito-identity.amazonaws.com:aud": "<identity-pool-id>"},
      "ForAnyValue:StringLike": {"cognito-identity.amazonaws.com:amr": "authenticated"}
    }
  }
  ```
- Verify the identity pool has the correct provider configuration
- Check that the IAM role has the permissions needed by the application

SHOULD:
- Use role mapping rules to assign different roles based on claims
- Scope IAM permissions to the user's identity using `${cognito-identity.amazonaws.com:sub}`
- Enable unauthenticated access only if explicitly needed

MAY:
- Use Cognito User Pools with groups for role-based access control
- Implement custom authentication flows for advanced use cases

## Common Issues

- symptoms: "GetCredentialsForIdentity fails with 'NotAuthorizedException'"
  diagnosis: "The identity pool doesn't recognize the provider token, or the pool is not configured for the provider."
  resolution: "Verify the provider is configured in the identity pool and the token is valid."

- symptoms: "Authenticated user gets unauthenticated role"
  diagnosis: "The token is not being passed correctly, or the amr claim doesn't include 'authenticated'."
  resolution: "Ensure the login map includes the provider token when calling GetId and GetCredentialsForIdentity."

- symptoms: "Credentials received but API calls are denied"
  diagnosis: "The Cognito role's IAM policy doesn't include the required permissions."
  resolution: "Add the required permissions to the authenticated role's IAM policy."

## Output Format

```yaml
root_cause: "web_identity — <specific_cause>"
evidence:
  - type: identity_pool
    content: "<pool configuration>"
  - type: role_mapping
    content: "<authenticated/unauthenticated role ARNs>"
  - type: trust_policy
    content: "<Cognito role trust policy>"
severity: MEDIUM
mitigation:
  immediate: "Fix identity pool configuration or role trust policy"
  long_term: "Implement proper role mapping and least-privilege policies"
```


## Safety Ratings

safety_ratings:
  - "Web identity federation configuration (Cognito identity pool and role mapping changes): YELLOW — Changes to identity pool configuration or role mappings affect how federated users receive AWS credentials"

## Escalation Conditions

escalation_conditions:
  - "Escalate to a human if the identity pool is in a production environment"
  - "Escalate if the change involves enabling unauthenticated access"
  - "Escalate if the role mapping change could grant elevated permissions to social identity users"

## Rollback

If remediation makes things worse:
1. Document the change that was made
2. Restore the original identity pool role mapping using `aws cognito-identity set-identity-pool-roles --identity-pool-id <pool-id> --roles file://original-roles.json`, and restore the original trust policy using `aws iam update-assume-role-policy --role-name <role> --policy-document file://original-trust-policy.json`
3. Verify access is restored to pre-change state

## Data Sensitivity

- `aws cognito-identity describe-identity-pool` returns pool configuration including provider settings — classification: MEDIUM
- `aws cognito-identity get-identity-pool-roles` reveals role mappings and authenticated/unauthenticated role ARNs — classification: MEDIUM
- Cognito user tokens may contain PII (email, name) from social identity providers — classification: HIGH

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting AdministratorAccess to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- Always identify the SPECIFIC denied action from CloudTrail before suggesting a fix
- Always scope fixes to the minimum required permissions (least privilege)

## Cleanup

- No temporary resources are created during web identity federation troubleshooting

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
