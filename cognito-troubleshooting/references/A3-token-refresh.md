---
title: "A3 — Cognito Token Refresh Issues"
description: "Diagnose token refresh failures and token expiry problems"
status: active
severity: HIGH
triggers:
  - "token expired"
  - "refresh token"
  - "NotAuthorizedException"
  - "invalid refresh token"
  - "token refresh failed"
owner: devops-agent
objective: "Identify why token refresh is failing and restore session continuity"
context: "Token refresh failures occur when refresh tokens expire, are revoked, the user is disabled, or the app client configuration changes. ID tokens are always 1 hour. Access tokens are configurable (5min-24hrs). Refresh tokens are configurable (1hr-3650 days)."
---

## Phase 1 — Triage

MUST:
- Identify which token is failing (ID, access, or refresh)
- Check app client token configuration: `aws cognito-idp describe-user-pool-client --user-pool-id <pool-id> --client-id <client-id> --query 'UserPoolClient.{AccessTokenValidity:AccessTokenValidity,IdTokenValidity:IdTokenValidity,RefreshTokenValidity:RefreshTokenValidity,TokenValidityUnits:TokenValidityUnits}'`
- Check if the user is still enabled: `aws cognito-idp admin-get-user --user-pool-id <pool-id> --username <username>`
- Check if REFRESH_TOKEN_AUTH flow is enabled: `aws cognito-idp describe-user-pool-client --user-pool-id <pool-id> --client-id <client-id> --query 'UserPoolClient.ExplicitAuthFlows'`

SHOULD:
- Check if the refresh token was revoked: `aws cognito-idp admin-list-user-auth-events --user-pool-id <pool-id> --username <username> --max-results 10`
- Check if GlobalSignOut was called for the user
- Verify the app client ID matches the one used to issue the original tokens

MAY:
- Decode the JWT token to check expiry and claims (use jwt.io or local decode)
- Check if the app client was recreated (new client ID invalidates old tokens)

## Phase 2 — Remediate

MUST:
- If refresh token expired: user must re-authenticate
- If REFRESH_TOKEN_AUTH not enabled: add it to the app client
- If user disabled: re-enable if appropriate

SHOULD:
- Adjust token validity settings to match application needs
- Implement proactive token refresh before expiry in the application

MAY:
- Implement token revocation handling in the application

## Common Issues

- symptoms: "NotAuthorizedException on InitiateAuth with REFRESH_TOKEN_AUTH"
  diagnosis: "Refresh token has expired or been revoked."
  resolution: "Check refresh token validity period. User must re-authenticate to get new tokens."

- symptoms: "Tokens expire too quickly"
  diagnosis: "Default access token validity is 1 hour. ID token is always 1 hour."
  resolution: "Increase access token validity (up to 24 hours) or implement proper refresh logic."

- symptoms: "Token refresh works but user attributes are stale"
  diagnosis: "ID token contains user attributes at time of issuance. Refresh gives new tokens with current attributes."
  resolution: "Use refresh token to get updated ID tokens with current attributes."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Add REFRESH_TOKEN_AUTH flow | YELLOW | App client config change; recoverable |
| Adjust token validity settings | YELLOW | Configuration change; recoverable |
| Re-enable disabled user | YELLOW | State change; can be reversed |

## Escalation Conditions

- Production user pool app client token validity changes
- Changes affecting all active user sessions
- Identity pool role mapping changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-user-pool-client` (token config) | MEDIUM | Token validity settings |
| `admin-get-user` | HIGH | User attributes and PII |
| `admin-list-user-auth-events` | HIGH | Auth history with IP addresses |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production
- NEVER suggest disabling advanced security features
- NEVER log or expose JWT tokens in diagnostic output

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Token validity change | Revert via `update-user-pool-client` with previous values |
| Auth flow enablement | Remove the flow via `update-user-pool-client` |
| User re-enable | Disable again via `admin-disable-user` |

## Output Format

```yaml
root_cause: "token_refresh_failure — <specific_cause>"
evidence:
  - type: token_config
    content: "<app client token validity settings>"
  - type: user_status
    content: "<user enabled/disabled status>"
severity: HIGH
mitigation:
  immediate: "Fix token configuration or re-authenticate user"
  long_term: "Implement proactive token refresh and proper session management"
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
