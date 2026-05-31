---
title: "C2 — Cognito User Pool Authorizer Issues"
description: "Diagnose Cognito user pool authorizer failures including token validation and configuration"
status: active
severity: HIGH
triggers:
  - "Cognito"
  - "user pool"
  - "JWT"
  - "ID token"
  - "access token"
  - "Cognito authorizer"
owner: devops-agent
objective: "Resolve Cognito user pool authorizer issues by validating token format, user pool configuration, and authorizer settings"
context: "Cognito user pool authorizers validate JWT tokens (ID or access tokens) issued by Amazon Cognito. The authorizer checks token signature, expiration, issuer (iss), and optionally audience (aud) and scopes. Common failures include expired tokens, wrong token type, misconfigured user pool ID, or missing required scopes."
---

## Phase 1 — Triage

MUST:
- Check authorizer configuration: `aws apigateway get-authorizer --rest-api-id <api-id> --authorizer-id <auth-id>`
- Verify user pool exists and is active: `aws cognito-idp describe-user-pool --user-pool-id <pool-id>`
- Check authorizer provider ARNs match the user pool
- Decode the JWT token to check claims (iss, aud, exp, token_use): `echo '<token>' | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool`
- Check execution logs: `aws logs filter-log-events --log-group-name API-Gateway-Execution-Logs_<api-id>/<stage> --filter-pattern "Unauthorized" --start-time <epoch-ms>`

SHOULD:
- Verify token type matches authorizer expectation (ID token vs access token)
- Check if required OAuth scopes are configured on the method: `aws apigateway get-method --rest-api-id <api-id> --resource-id <resource-id> --http-method <method> --query 'authorizationScopes'`
- Verify the app client ID is in the token audience claim

MAY:
- Check user pool app client settings: `aws cognito-idp describe-user-pool-client --user-pool-id <pool-id> --client-id <client-id>`
- Verify token has not been revoked

## Phase 2 — Remediate

MUST:
- Ensure authorizer providerARNs contains the correct user pool ARN
- Verify the token is not expired (check exp claim)
- Use the correct token type: ID token for identity claims, access token for scopes
- Ensure the Authorization header contains the full token (Bearer prefix may or may not be needed depending on configuration)

SHOULD:
- Configure authorization scopes on methods that require specific OAuth scopes
- Use access tokens (not ID tokens) when scope validation is needed
- Set appropriate token expiration in Cognito user pool settings

MAY:
- Implement token refresh logic in the client application
- Use Cognito hosted UI for standardized authentication flow

## Common Issues

- symptoms: "401 Unauthorized with valid Cognito token"
  diagnosis: "Token is expired, wrong type (ID vs access), or user pool ARN mismatch."
  resolution: "Verify token expiration, type, and that authorizer providerARNs matches the user pool."

- symptoms: "401 when using access token with scopes"
  diagnosis: "Method authorizationScopes do not match the scopes in the access token."
  resolution: "Ensure the access token contains the required scopes and method scopes are configured correctly."

- symptoms: "401 after Cognito user pool domain change"
  diagnosis: "Token issuer (iss) claim no longer matches the expected issuer."
  resolution: "Obtain new tokens from the updated user pool. Old tokens are invalid."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
```

## Escalation Conditions

- API serves production traffic from external or internal consumers
- Fix requires redeploying a production stage
- Resolution involves modifying authorizer configuration
- Issue affects multiple APIs or is account-level throttling
- Custom domain name or certificate changes are required

## Data Sensitivity

MEDIUM - API keys and usage plan configurations control access to APIs. Authorizer configurations (Lambda authorizer code, Cognito pool settings) are security-sensitive. Access logs may contain request parameters, headers, and client IPs. Mapping templates may expose internal backend URLs and data transformation logic.

## Prohibited Actions

- NEVER suggest deleting a production stage - this immediately removes the API endpoint
- NEVER suggest removing all authorizers from an API that requires authentication
- NEVER recommend disabling API key requirements on usage-plan-protected endpoints
- NEVER suggest modifying throttling limits to unlimited on a production API
- NEVER recommend deleting and recreating a custom domain name - this causes DNS propagation downtime

## Phase 3 - Rollback

1. If a deployment was made, redeploy the previous working deployment to the stage: `aws apigateway create-deployment --rest-api-id <id> --stage-name <stage>`
2. If authorizer configuration was changed, restore the original authorizer settings and redeploy the stage
3. If integration settings were changed, restore original integration configuration and redeploy
4. If custom domain or base path mapping was changed, restore original mapping configuration
5. If CORS or method configuration was changed, restore original settings and redeploy the API
6. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "cognito_authorizer — <specific_cause>"
evidence:
  - type: authorizer_config
    content: "<authorizer provider ARNs and settings>"
  - type: token_claims
    content: "<decoded JWT claims (iss, aud, exp, token_use)>"
  - type: user_pool
    content: "<user pool configuration>"
severity: HIGH
mitigation:
  immediate: "Fix token type, expiration, or authorizer configuration"
  long_term: "Implement token refresh and proper scope management"
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
  - "NEVER suggest removing authorizers to fix access"
  - "NEVER suggest disabling throttling"
  - "NEVER suggest open CORS for production APIs"
