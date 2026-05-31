---
title: "D3 — IAM and OIDC Authorization"
description: "Diagnose AppSync IAM and OpenID Connect authorization failures"
status: active
severity: HIGH
triggers:
  - "IAM auth"
  - "SigV4"
  - "OIDC auth"
  - "OpenID Connect"
  - "IAM unauthorized"
  - "OIDC token"
  - "Lambda authorizer"
owner: devops-agent
objective: "Identify and resolve IAM, OIDC, and Lambda authorization issues in AppSync"
context: "AWS_IAM auth uses SigV4 signed requests. OPENID_CONNECT validates tokens from external OIDC providers. AWS_LAMBDA uses a custom Lambda function for authorization. Common issues include missing IAM permissions, incorrect SigV4 signing, wrong OIDC issuer URL, and Lambda authorizer errors."
---

## Phase 1 — Triage

MUST:
- Check the API auth configuration: `aws appsync get-graphql-api --api-id <api-id>`
- For IAM: verify the caller has `appsync:GraphQL` permission on the API ARN
- For IAM: check the IAM policy allows the specific GraphQL operations
- For OIDC: verify the issuer URL is correct and accessible
- For Lambda: check the Lambda authorizer function: `aws lambda get-function --function-name <auth-function>`

SHOULD:
- For IAM: check if the request is properly SigV4 signed (common issue with SDK configuration)
- For OIDC: decode the token and verify the issuer (iss) claim matches the configured issuer
- For Lambda: check Lambda authorizer logs for errors
- Check if multi-auth directives (@aws_iam, @aws_oidc) are on the target types

MAY:
- For IAM: use CloudTrail to check for denied API calls
- For OIDC: verify the OIDC provider's JWKS endpoint is accessible
- For Lambda: test the authorizer function independently

## Phase 2 — Remediate

MUST:
- For IAM: add `appsync:GraphQL` permission with the correct API ARN and field-level resource
- For OIDC: fix the issuer URL to match the token's iss claim exactly
- For Lambda: fix the authorizer function errors

SHOULD:
- For IAM: use resource-level permissions (arn:aws:appsync:<region>:<account>:apis/<api-id>/types/*/fields/*)
- For OIDC: configure the correct client ID (aud claim) if required
- For Lambda: implement proper error handling and return the correct authorization response format

MAY:
- Use IAM policy conditions for fine-grained access control
- Cache Lambda authorizer results to reduce invocations

## Common Issues

- symptoms: "UnauthorizedException with IAM auth"
  diagnosis: "IAM policy missing appsync:GraphQL action or incorrect resource ARN."
  resolution: "Add appsync:GraphQL permission with the correct API ARN."

- symptoms: "OIDC token rejected"
  diagnosis: "Token issuer (iss claim) doesn't match the configured OIDC issuer URL."
  resolution: "Ensure the issuer URL in AppSync exactly matches the iss claim in the token."

- symptoms: "Lambda authorizer returns 401"
  diagnosis: "Lambda function returns isAuthorized: false or throws an error."
  resolution: "Check Lambda function logs. Ensure it returns { isAuthorized: true, ... } for valid requests."

## Output Format

```yaml
root_cause: "iam_oidc_auth_failure — <specific_cause>"
evidence:
  - type: auth_config
    content: "<auth mode configuration>"
  - type: iam_policy
    content: "<IAM policy or OIDC/Lambda config>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions, OIDC issuer, or Lambda authorizer"
  long_term: "Implement least-privilege IAM policies and token validation"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Add appsync:GraphQL permission | YELLOW | IAM policy change |
| Fix OIDC issuer URL | YELLOW | Auth configuration change |
| Fix Lambda authorizer errors | YELLOW | Lambda code change |
| Use resource-level IAM permissions | YELLOW | IAM policy change |
| Configure OIDC client ID | YELLOW | Auth configuration change |
| Cache Lambda authorizer results | GREEN | Performance improvement |

## Escalation Conditions

- Fix requires modifying production resolver
- Fix requires changing auth mode

## Data Sensitivity

- **IAM policies**: appsync:GraphQL permissions — INTERNAL
- **OIDC tokens**: JWT tokens with user claims — CONFIDENTIAL
- **Lambda authorizer**: Custom authorization logic — INTERNAL

## Prohibited Actions

- NEVER suggest removing authorization to fix access denied
- NEVER grant appsync:GraphQL on * resource without field-level restrictions
- NEVER log JWT tokens or authorization headers

## Phase 3 — Rollback

1. Revert IAM policy changes to previous version
2. Revert OIDC issuer URL configuration
3. Revert Lambda authorizer function to previous version
4. Verify all auth modes are working correctly
5. Test authentication with each configured auth mode

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
  - "NEVER suggest removing authentication from APIs"
  - "NEVER suggest API_KEY auth for production"
  - "NEVER suggest disabling logging"
