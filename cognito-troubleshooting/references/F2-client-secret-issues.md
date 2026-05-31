---
title: "F2 — App Client Secret Issues"
description: "Diagnose issues caused by app client secret misconfiguration"
status: active
severity: HIGH
triggers:
  - "client secret"
  - "SecretHash"
  - "Unable to verify secret hash"
  - "client secret SPA"
  - "Amplify secret"
owner: devops-agent
objective: "Identify and resolve app client secret compatibility issues"
context: "App clients with secrets require a SECRET_HASH parameter in API calls. Browser-based and mobile apps cannot securely store secrets. Using a secret-enabled client with SPA/mobile SDKs causes authentication failures."
---

## Phase 1 — Triage

MUST:
- Check if the app client has a secret: `aws cognito-idp describe-user-pool-client --user-pool-id <pool-id> --client-id <client-id> --query 'UserPoolClient.ClientSecret'`
- Identify the application type (SPA, mobile, server-side)
- Check the exact error message

SHOULD:
- Verify if the application is computing SECRET_HASH correctly (for server-side apps)
- Check if the SDK being used supports client secrets

MAY:
- List all app clients to find one without a secret: `aws cognito-idp list-user-pool-clients --user-pool-id <pool-id>`

## Phase 2 — Remediate

MUST:
- For SPA/mobile: create a new app client WITHOUT a secret
- For server-side: compute SECRET_HASH = Base64(HMAC_SHA256(clientSecret, username + clientId))
- Update the application to use the correct client ID

SHOULD:
- Migrate users/sessions to the new app client if switching
- Document which app clients are for which application types

MAY:
- Remove the old secret-enabled client if no longer needed

## Common Issues

- symptoms: "Unable to verify secret hash for client"
  diagnosis: "App client has a secret but the API call doesn't include SECRET_HASH."
  resolution: "For SPA/mobile: create a client without secret. For server: compute and include SECRET_HASH."

- symptoms: "Amplify throws error about client secret"
  diagnosis: "Amplify SDK does not support app clients with secrets."
  resolution: "Create a new app client without a secret and update Amplify configuration."

- symptoms: "SECRET_HASH computation fails"
  diagnosis: "Incorrect HMAC computation. The message must be username + clientId."
  resolution: "Compute: Base64(HMAC_SHA256(clientSecret, username + clientId)). Use the username, not email."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create new app client without secret | YELLOW | New resource; old client unaffected |
| Compute SECRET_HASH | GREEN | Read-only computation; no state change |
| Update application client ID | YELLOW | Application config change; recoverable |

## Escalation Conditions

- Production user pool app client changes
- Application configuration changes affecting active users
- Identity pool role mapping changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-user-pool-client` (ClientSecret) | HIGH | Contains client secret |
| `list-user-pool-clients` | LOW | Client IDs and names only |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production
- NEVER suggest disabling advanced security features
- NEVER expose client secrets in logs or diagnostic output

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| New app client creation | Delete new client; revert app to old client ID |
| Application client ID update | Revert application configuration |

## Output Format

```yaml
root_cause: "client_secret_issue — <specific_cause>"
evidence:
  - type: client_config
    content: "<client secret presence and app type>"
severity: HIGH
mitigation:
  immediate: "Create appropriate app client or fix SECRET_HASH computation"
  long_term: "Maintain separate app clients for public and confidential applications"
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
