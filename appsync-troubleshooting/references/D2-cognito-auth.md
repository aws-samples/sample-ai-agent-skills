---
title: "D2 — Cognito User Pool Authorization"
description: "Diagnose AppSync Cognito User Pool authorization failures"
status: active
severity: HIGH
triggers:
  - "Cognito auth"
  - "Cognito user pool"
  - "JWT token"
  - "token expired"
  - "Cognito unauthorized"
  - "user pool"
owner: devops-agent
objective: "Identify and resolve Cognito User Pool authorization issues in AppSync"
context: "AMAZON_COGNITO_USER_POOLS auth mode validates JWT tokens from Cognito. Common issues include expired tokens, wrong user pool ID, missing groups for group-based access, incorrect app client configuration, and token validation failures. Tokens are passed in the Authorization header."
---

## Phase 1 — Triage

MUST:
- Check the API Cognito auth configuration: `aws appsync get-graphql-api --api-id <api-id>`
- Verify the user pool exists: `aws cognito-idp describe-user-pool --user-pool-id <pool-id>`
- Check if the app client ID matches: `aws cognito-idp describe-user-pool-client --user-pool-id <pool-id> --client-id <client-id>`
- Decode the JWT token to check expiration, issuer, and audience (use jwt.io or `echo <token> | cut -d. -f2 | base64 -d`)
- Check CloudWatch Logs for specific token validation errors

SHOULD:
- Verify the token issuer matches the user pool URL (https://cognito-idp.<region>.amazonaws.com/<pool-id>)
- Check if group-based access control is configured and the user belongs to the required group
- Verify the app client allows the required OAuth scopes

MAY:
- Check if the user pool has a custom domain that might affect token issuer
- Verify token revocation settings

## Phase 2 — Remediate

MUST:
- Ensure the user pool ID in AppSync config matches the actual user pool
- Ensure the app client ID is correct (if configured in AppSync)
- Fix expired tokens by refreshing or re-authenticating

SHOULD:
- Configure appropriate token expiration times in the user pool
- Use refresh tokens for long-lived sessions
- Add @aws_cognito_user_pools directive with cognito_groups for group-based access

MAY:
- Implement token refresh logic in the client application
- Use Cognito Identity Pools for IAM-based access patterns

## Common Issues

- symptoms: "UnauthorizedException with valid-looking JWT"
  diagnosis: "Token is from a different user pool than configured in AppSync."
  resolution: "Verify the user pool ID in AppSync matches the token issuer."

- symptoms: "Access denied for specific operations"
  diagnosis: "Schema uses @aws_cognito_user_pools(cognito_groups: ['Admin']) and user isn't in the group."
  resolution: "Add the user to the required Cognito group."

- symptoms: "Token expired errors"
  diagnosis: "ID token has expired (default 1 hour). Client isn't refreshing tokens."
  resolution: "Implement token refresh using the refresh token. Increase token validity if needed."

## Output Format

```yaml
root_cause: "cognito_auth_failure — <specific_cause>"
evidence:
  - type: auth_config
    content: "<user pool ID and app client configuration>"
  - type: token_details
    content: "<decoded JWT claims>"
severity: HIGH
mitigation:
  immediate: "Fix user pool configuration or token refresh"
  long_term: "Implement proper token lifecycle management"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure user pool ID matches config | GREEN | Configuration verification |
| Ensure app client ID is correct | GREEN | Configuration verification |
| Fix expired tokens by refreshing | GREEN | Client-side fix |
| Configure token expiration times | YELLOW | User pool configuration change |
| Add @aws_cognito_user_pools directive | YELLOW | Schema change |
| Use Cognito Identity Pools | YELLOW | Auth architecture change |

## Escalation Conditions

- Fix requires modifying production resolver
- Fix requires changing auth mode

## Data Sensitivity

- **User pool config**: Pool ID, app client settings — INTERNAL
- **JWT tokens**: User identity and claims — CONFIDENTIAL
- **Cognito groups**: User group memberships — INTERNAL

## Prohibited Actions

- NEVER suggest removing authorization to fix access denied
- NEVER log JWT tokens or user credentials
- NEVER modify Cognito user pool settings without understanding impact on all clients

## Phase 3 — Rollback

1. Revert AppSync auth configuration to previous user pool settings
2. Revert schema directive changes
3. Revert Cognito user pool token expiration settings
4. Verify all clients can authenticate after rollback
5. Test group-based access control if it was modified

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
