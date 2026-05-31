---
title: "A2 — Cognito Sign-In Failures"
description: "Diagnose user sign-in failures in Cognito user pools"
status: active
severity: HIGH
triggers:
  - "sign-in failed"
  - "login error"
  - "NotAuthorizedException"
  - "UserNotFoundException"
  - "UserNotConfirmedException"
owner: devops-agent
objective: "Identify why user sign-in is failing and resolve authentication issues"
context: "Sign-in failures can be caused by incorrect credentials, unconfirmed users, disabled users, incorrect auth flow configuration, app client mismatch, or pre-authentication Lambda trigger rejections."
---

## Phase 1 — Triage

MUST:
- Get the exact error message and auth flow being used
- Check user status: `aws cognito-idp admin-get-user --user-pool-id <pool-id> --username <username>`
- Check app client auth flows: `aws cognito-idp describe-user-pool-client --user-pool-id <pool-id> --client-id <client-id> --query 'UserPoolClient.ExplicitAuthFlows'`
- Check if pre-authentication trigger exists: `aws cognito-idp describe-user-pool --user-pool-id <pool-id> --query 'UserPool.LambdaConfig.PreAuthentication'`
- Check user auth events: `aws cognito-idp admin-list-user-auth-events --user-pool-id <pool-id> --username <username> --max-results 10`

SHOULD:
- Check if the user is in FORCE_CHANGE_PASSWORD status
- Check pre-authentication Lambda logs: `aws logs filter-log-events --log-group-name /aws/lambda/<pre-auth-function> --start-time <epoch-ms>`
- Check if MFA is required: `aws cognito-idp get-user-pool-mfa-config --user-pool-id <pool-id>`

MAY:
- Check advanced security risk evaluation if enabled
- Check CloudWatch sign-in metrics

## Phase 2 — Remediate

MUST:
- Confirm unconfirmed users: `aws cognito-idp admin-confirm-sign-up --user-pool-id <pool-id> --username <username>`
- Enable the correct auth flow on the app client if missing
- Reset password for locked-out users: `aws cognito-idp admin-set-user-password --user-pool-id <pool-id> --username <username> --password <temp-password> --no-permanent`

SHOULD:
- Fix pre-authentication Lambda if it is incorrectly rejecting users
- Enable the user if disabled: `aws cognito-idp admin-enable-user --user-pool-id <pool-id> --username <username>`

MAY:
- Review and adjust adaptive authentication risk thresholds

## Common Issues

- symptoms: "NotAuthorizedException: Incorrect username or password"
  diagnosis: "Credentials are wrong, or the user does not exist and the pool is configured to hide this."
  resolution: "Verify credentials. Check if the pool prevents user existence errors (PreventUserExistenceErrors setting)."

- symptoms: "UserNotConfirmedException"
  diagnosis: "User signed up but never confirmed their email/phone."
  resolution: "Resend confirmation code or admin-confirm the user."

- symptoms: "NotAuthorizedException: User is disabled"
  diagnosis: "The user account has been administratively disabled."
  resolution: "Re-enable with admin-enable-user if appropriate."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| `admin-confirm-sign-up` | YELLOW | State-changing; user status change is recoverable |
| Enable auth flow on app client | YELLOW | Configuration change; recoverable |
| `admin-set-user-password` | YELLOW | Credential change; user can reset again |
| `admin-enable-user` | YELLOW | Re-enables user; can be disabled again |
| Fix pre-authentication Lambda | YELLOW | Code change; previous version restorable |

## Escalation Conditions

- Production user pool app client auth flow changes
- Bulk user status changes (confirming/enabling multiple users)
- Pre-authentication Lambda modifications in production
- Identity pool role mapping changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `admin-get-user` | HIGH | Returns user attributes and PII |
| `describe-user-pool-client` | MEDIUM | Client configuration details |
| `admin-list-user-auth-events` | HIGH | Authentication history with IP addresses |
| `filter-log-events` (Lambda logs) | HIGH | May contain user identifiers |
| `get-user-pool-mfa-config` | LOW | Pool-level MFA settings |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production without a rollback plan
- NEVER suggest disabling advanced security features
- NEVER log or expose user credentials
- NEVER disable a user pool's PreventUserExistenceErrors setting

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| `admin-confirm-sign-up` | Cannot unconfirm; document the action |
| Auth flow change | Revert app client auth flows via `update-user-pool-client` |
| `admin-set-user-password` | User can reset password through normal flow |
| `admin-enable-user` | Disable user again via `admin-disable-user` |
| Pre-auth Lambda fix | Redeploy previous Lambda version |

## Output Format

```yaml
root_cause: "sign_in_failure — <specific_cause>"
evidence:
  - type: user_status
    content: "<admin-get-user result>"
  - type: auth_events
    content: "<recent auth events>"
severity: HIGH
mitigation:
  immediate: "Fix user status or auth flow configuration"
  long_term: "Implement proper error handling and user status management"
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
