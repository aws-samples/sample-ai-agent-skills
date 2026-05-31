---
title: "E1 — Hosted UI Errors"
description: "Diagnose Cognito hosted UI loading and configuration errors"
status: active
severity: MEDIUM
triggers:
  - "hosted UI"
  - "login page error"
  - "hosted UI not loading"
  - "OAuth error"
  - "redirect_mismatch"
owner: devops-agent
objective: "Identify and fix hosted UI configuration and display issues"
context: "The Cognito hosted UI provides a managed sign-in/sign-up page. Issues arise from domain misconfiguration, incorrect callback URLs, missing OAuth settings, or CSS customization problems. Customization is limited to CSS (200 KB) and logo (100 KB)."
---

## Phase 1 — Triage

MUST:
- Check domain configuration: `aws cognito-idp describe-user-pool-domain --domain <domain>`
- Check app client OAuth settings: `aws cognito-idp describe-user-pool-client --user-pool-id <pool-id> --client-id <client-id> --query 'UserPoolClient.{AllowedOAuthFlows:AllowedOAuthFlows,AllowedOAuthScopes:AllowedOAuthScopes,CallbackURLs:CallbackURLs,LogoutURLs:LogoutURLs}'`
- Verify callback URLs match the application's redirect URI exactly
- Check if OAuth flows are enabled (code, implicit, or client_credentials)

SHOULD:
- Check UI customization: `aws cognito-idp get-ui-customization --user-pool-id <pool-id> --client-id <client-id>`
- Verify the domain is accessible by visiting the hosted UI URL
- Check if identity providers are configured on the app client

MAY:
- Check browser console for JavaScript errors on the hosted UI page
- Verify DNS resolution for custom domains

## Phase 2 — Remediate

MUST:
- Fix callback URLs to match exactly (including trailing slashes)
- Enable required OAuth flows and scopes on the app client
- Create or fix the domain configuration

SHOULD:
- Fix CSS customization if the UI renders incorrectly
- Add all required identity providers to the app client

MAY:
- Consider building a custom UI if hosted UI limitations are blocking

## Common Issues

- symptoms: "redirect_mismatch error after login"
  diagnosis: "The callback URL in the authorization request doesn't match any configured callback URL."
  resolution: "Add the exact callback URL to the app client. URLs are case-sensitive and must match exactly."

- symptoms: "Hosted UI shows blank page"
  diagnosis: "Domain is not configured or CSS customization has errors."
  resolution: "Verify domain exists. Remove custom CSS to test with defaults."

- symptoms: "Identity providers not showing on hosted UI"
  diagnosis: "Identity providers are not added to the app client's supported providers."
  resolution: "Add the identity providers to the app client's SupportedIdentityProviders list."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix callback URLs | YELLOW | App client config change; recoverable |
| Enable OAuth flows/scopes | YELLOW | Configuration change; recoverable |
| Fix domain configuration | YELLOW | Domain change; recoverable |
| Fix CSS customization | GREEN | Visual change only; no security impact |

## Escalation Conditions

- Production user pool domain changes
- OAuth flow changes affecting all users
- Callback URL changes in production

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-user-pool-domain` | LOW | Domain configuration |
| `describe-user-pool-client` (OAuth) | MEDIUM | OAuth configuration with callback URLs |
| `get-ui-customization` | LOW | CSS and logo configuration |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production
- NEVER suggest disabling advanced security features

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Callback URL change | Revert via `update-user-pool-client` |
| OAuth flow change | Revert via `update-user-pool-client` |
| Domain change | Delete and recreate domain with previous settings |
| CSS customization | Revert via `set-ui-customization` |

## Output Format

```yaml
root_cause: "hosted_ui_error — <specific_cause>"
evidence:
  - type: domain_config
    content: "<domain configuration>"
  - type: oauth_config
    content: "<app client OAuth settings>"
severity: MEDIUM
mitigation:
  immediate: "Fix OAuth configuration or callback URLs"
  long_term: "Implement URL validation and monitoring for hosted UI"
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
