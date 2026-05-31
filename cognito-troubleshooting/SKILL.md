---
name: cognito-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Cognito problems
  by analyzing user pool configurations, identity pool federation,
  token issues, and following structured runbooks. Activate when:
  sign-up/sign-in failures, token refresh errors, custom auth flow
  issues, SAML/OIDC federation problems, Lambda trigger failures,
  hosted UI errors, MFA configuration issues, app client
  misconfiguration, user migration problems, custom domain issues,
  or the user says something is wrong with Cognito without naming
  specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Cognito User Pools,
  Cognito Identity, IAM, Lambda, CloudWatch, and optionally
  SNS and SES permissions.
---

# Cognito Diagnostics

## When to use

Any Cognito investigation where the console alone is insufficient — authentication flow failures, federation debugging, token analysis, Lambda trigger troubleshooting, hosted UI issues, MFA problems, or app client configuration errors.

## Investigation workflow

### Step 1 — Collect and triage

```
aws cognito-idp describe-user-pool --user-pool-id <pool-id>
aws cognito-idp describe-user-pool-client --user-pool-id <pool-id> --client-id <client-id>
aws cognito-idp list-user-pool-clients --user-pool-id <pool-id>
aws cognito-idp get-user-pool-mfa-config --user-pool-id <pool-id>
aws cognito-idp list-identity-providers --user-pool-id <pool-id>
aws cognito-identity describe-identity-pool --identity-pool-id <identity-pool-id>
```

### Step 2 — Domain deep dive

```
aws cognito-idp describe-user-pool-domain --domain <domain>
aws cognito-idp list-groups --user-pool-id <pool-id>
aws cognito-idp admin-get-user --user-pool-id <pool-id> --username <username>
aws lambda get-function --function-name <trigger-function>
aws cloudwatch get-metric-statistics --namespace AWS/Cognito --metric-name SignInSuccesses ...
aws logs filter-log-events --log-group-name /aws/lambda/<trigger-function>
```

Read `references/cognito-guardrails.md` before concluding on any Cognito issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-user-pool` | Pool configuration, policies, triggers |
| `describe-user-pool-client` | App client settings, OAuth config |
| `get-user-pool-mfa-config` | MFA configuration |
| `list-identity-providers` | SAML/OIDC federation providers |
| `describe-identity-pool` | Identity pool roles, federation |
| `admin-get-user` | User attributes, status, MFA |
| `admin-list-user-auth-events` | User authentication history |
| `describe-user-pool-domain` | Hosted UI domain config |
| `get-identity-pool-roles` | Identity pool role mappings |

## Gotchas: Cognito

- User pool vs identity pool: these are DIFFERENT services. User pools handle authentication (sign-up/sign-in). Identity pools handle authorization (temporary AWS credentials). They can work together or independently.
- Token expiry: ID token = 1 hour, access token = 1 hour (configurable 5min-24hrs), refresh token = 30 days (configurable 1hr-3650 days). Refresh tokens are the only way to get new ID/access tokens without re-authentication.
- Lambda triggers have a 5-second timeout. Complex logic in triggers causes authentication timeouts. Triggers cannot be async.
- Hosted UI customization has limits: CSS max 200 KB, logo max 100 KB. Custom UI pages are not supported — only CSS/logo customization.
- SAML federation requires exact attribute mapping. The NameID and any mapped attributes must match exactly between the IdP and Cognito. Case sensitivity matters.
- App client secret is incompatible with SPA and mobile apps. JavaScript SDK and Amplify cannot securely store secrets. Use app clients WITHOUT secrets for public clients.
- Custom auth flow requires THREE Lambda triggers: Define Auth Challenge, Create Auth Challenge, and Verify Auth Challenge Response. Missing any one causes the flow to fail.
- User pool deletion is PERMANENT. All users, groups, and configurations are lost. There is no undo or recovery.
- Advanced security features (adaptive authentication, compromised credentials) cost extra per MAU. They are not included in the free tier.
- SMS MFA requires exiting the SNS sandbox for production. In sandbox mode, SMS is only sent to verified phone numbers.

### Token comparison

| Token | Lifetime | Contains | Use For |
|-------|----------|----------|---------|
| ID Token | 1 hour | User attributes, groups | Frontend identity |
| Access Token | 1 hour (5min-24hrs) | Scopes, groups | API authorization |
| Refresh Token | 30 days (1hr-3650d) | Opaque | Getting new tokens |

## Anti-hallucination rules

1. Always cite specific user pool configurations, app client settings, or API responses as evidence.
2. User pools and identity pools are different services. Never conflate authentication (user pools) with authorization (identity pools).
3. App client secrets cannot be used in browser/mobile apps. Never recommend secrets for SPA/mobile clients.
4. Lambda triggers have a 5-second timeout. Never suggest long-running operations in triggers.
5. Custom auth requires all three Lambda triggers. Never claim partial trigger setup works.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 28 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Auth Flows | A1-A4 | Sign-up failures, sign-in failures, token refresh, custom auth |
| B — Federation | B1-B3 | SAML, OIDC, identity pool |
| C — Triggers | C1-C3 | Pre-sign-up, pre-authentication, post-confirmation |
| D — MFA | D1-D2 | SMS MFA, TOTP MFA |
| E — Hosted UI | E1-E2 | UI errors, custom domain |
| F — App Client | F1-F2 | Client config, secret issues |
| G — User Management | G1-G2 | Migration, import |
| H — Advanced | H1-H2 | Adaptive auth, compromised credentials |
| Z — Catch-All | Z1 | General troubleshooting |
