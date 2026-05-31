---
title: "B3 — Identity Pool Federation Issues"
description: "Diagnose Cognito identity pool role mapping and credential vending failures"
status: active
severity: HIGH
triggers:
  - "identity pool"
  - "federated identity"
  - "GetCredentialsForIdentity"
  - "NotAuthorizedException identity"
  - "AWS credentials from Cognito"
owner: devops-agent
objective: "Identify why identity pool is failing to vend AWS credentials and fix role mapping"
context: "Identity pools exchange tokens (from user pools, SAML, OIDC, or social providers) for temporary AWS credentials via STS. Failures occur due to incorrect role mapping, trust policy issues, or token validation failures."
---

## Phase 1 — Triage

MUST:
- Check identity pool configuration: `aws cognito-identity describe-identity-pool --identity-pool-id <identity-pool-id>`
- Check role mappings: `aws cognito-identity get-identity-pool-roles --identity-pool-id <identity-pool-id>`
- Verify the IAM roles' trust policies allow the identity pool: `aws iam get-role --role-name <auth-role-name> --query 'Role.AssumeRolePolicyDocument'`
- Check if the identity pool allows unauthenticated access if needed

SHOULD:
- Verify the token provider is configured in the identity pool
- Check if rules-based role mapping is correctly configured
- Verify the authenticated role has the required permissions

MAY:
- Check CloudTrail for AssumeRoleWithWebIdentity failures
- Test with a known-good token

## Phase 2 — Remediate

MUST:
- Fix IAM role trust policy to include the identity pool as a trusted entity
- Configure correct role mappings (default, rules-based, or token-based)
- Ensure the identity provider is registered in the identity pool

SHOULD:
- Set a default authenticated role as fallback
- Use rules-based mapping for fine-grained role assignment

MAY:
- Enable CloudTrail logging for identity pool API calls

## Common Issues

- symptoms: "NotAuthorizedException: Unauthenticated access is not supported"
  diagnosis: "Identity pool has unauthenticated access disabled but the request has no token."
  resolution: "Enable unauthenticated access or ensure the application sends a valid token."

- symptoms: "Credentials returned but access denied on AWS resources"
  diagnosis: "The IAM role mapped to the identity has insufficient permissions."
  resolution: "Check the IAM role's permission policy. Add required actions and resources."

- symptoms: "Invalid identity pool configuration"
  diagnosis: "The identity provider (user pool, SAML, etc.) is not registered in the identity pool."
  resolution: "Add the identity provider to the identity pool's authentication providers."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix IAM role trust policy | YELLOW | IAM change; recoverable |
| Configure role mappings | YELLOW | Identity pool config change; recoverable |
| Register identity provider | YELLOW | Configuration change; recoverable |

## Escalation Conditions

- Production identity pool role mapping changes
- IAM role trust policy modifications
- Changes affecting AWS credential vending for all users

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-identity-pool` | MEDIUM | Pool configuration with provider details |
| `get-identity-pool-roles` | HIGH | IAM role ARNs and mapping rules |
| `get-role` (trust policy) | MEDIUM | IAM trust policy details |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production
- NEVER suggest disabling advanced security features
- NEVER grant overly broad IAM permissions to identity pool roles
- NEVER enable unauthenticated access without explicit approval

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Trust policy change | Revert IAM role trust policy to previous version |
| Role mapping change | Revert via `set-identity-pool-roles` |
| Provider registration | Remove provider from identity pool |

## Output Format

```yaml
root_cause: "identity_pool_failure — <specific_cause>"
evidence:
  - type: identity_pool_config
    content: "<identity pool configuration>"
  - type: role_mapping
    content: "<role mapping configuration>"
severity: HIGH
mitigation:
  immediate: "Fix role mapping or trust policy"
  long_term: "Implement proper role mapping strategy with default fallback"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-user-pool"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-users"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "admin-get-user"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling MFA for troubleshooting"
  - "NEVER expose user PII in investigation output"
  - "NEVER suggest removing password policies"
