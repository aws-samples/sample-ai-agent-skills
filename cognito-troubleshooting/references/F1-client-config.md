---
title: "F1 — App Client Configuration Issues"
description: "Diagnose Cognito app client misconfiguration"
status: active
severity: MEDIUM
triggers:
  - "app client"
  - "client configuration"
  - "auth flow not enabled"
  - "InvalidParameterException client"
owner: devops-agent
objective: "Identify and fix app client configuration issues"
context: "App clients control which auth flows, OAuth settings, and identity providers are available. Misconfiguration causes auth failures, missing features, or security issues."
---

## Phase 1 — Triage

MUST:
- Check app client configuration: `aws cognito-idp describe-user-pool-client --user-pool-id <pool-id> --client-id <client-id>`
- Verify enabled auth flows match the application's needs
- Check OAuth configuration (flows, scopes, callback URLs)
- Verify supported identity providers

SHOULD:
- Check if the client has a secret (incompatible with SPA/mobile)
- Verify token validity settings
- Check read/write attributes

MAY:
- Compare with other app clients in the same pool: `aws cognito-idp list-user-pool-clients --user-pool-id <pool-id>`

## Phase 2 — Remediate

MUST:
- Enable required auth flows (USER_SRP_AUTH, USER_PASSWORD_AUTH, CUSTOM_AUTH, etc.)
- Fix OAuth configuration
- Add required identity providers

SHOULD:
- Remove client secret for SPA/mobile clients (requires creating a new client)
- Configure appropriate token validity

MAY:
- Create separate app clients for different application types

## Common Issues

- symptoms: "InvalidParameterException: USER_SRP_AUTH flow not enabled"
  diagnosis: "The app client doesn't have the required auth flow enabled."
  resolution: "Update the app client to include the required ExplicitAuthFlows."

- symptoms: "OAuth redirect fails with invalid_client"
  diagnosis: "App client OAuth settings are misconfigured."
  resolution: "Enable OAuth flows, add callback URLs, and configure scopes."

- symptoms: "Cannot use Amplify with this app client"
  diagnosis: "App client has a secret, which Amplify cannot use."
  resolution: "Create a new app client without a secret for Amplify/SPA use."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable auth flows | YELLOW | App client config change; recoverable |
| Fix OAuth configuration | YELLOW | Configuration change; recoverable |
| Add identity providers | YELLOW | Configuration change; recoverable |

## Escalation Conditions

- Production user pool app client changes
- Auth flow changes affecting active users
- Identity pool role mapping changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-user-pool-client` | MEDIUM | Client configuration details |
| `list-user-pool-clients` | LOW | Client IDs and names only |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production
- NEVER suggest disabling advanced security features
- NEVER expose client secrets in diagnostic output

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Auth flow change | Revert via `update-user-pool-client` |
| OAuth config change | Revert via `update-user-pool-client` |
| Identity provider addition | Remove via `update-user-pool-client` |

## Output Format

```yaml
root_cause: "app_client_config_error — <specific_cause>"
evidence:
  - type: client_config
    content: "<app client configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix app client settings"
  long_term: "Document app client requirements per application type"
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
