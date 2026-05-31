---
title: "A1 — Cognito Sign-Up Failures"
description: "Diagnose user sign-up failures in Cognito user pools"
status: active
severity: HIGH
triggers:
  - "sign-up failed"
  - "registration error"
  - "cannot create user"
  - "UsernameExistsException"
  - "InvalidParameterException"
owner: devops-agent
objective: "Identify why user sign-up is failing and fix the user pool or client configuration"
context: "Sign-up failures can stem from user pool policies (password strength, required attributes), app client configuration, pre-sign-up Lambda trigger rejections, or email/phone verification settings. The error message usually indicates the specific cause."
---

## Phase 1 — Triage

MUST:
- Get the exact error message from the application or CloudWatch logs
- Check user pool password policy: `aws cognito-idp describe-user-pool --user-pool-id <pool-id> --query 'UserPool.Policies.PasswordPolicy'`
- Check required attributes: `aws cognito-idp describe-user-pool --user-pool-id <pool-id> --query 'UserPool.SchemaAttributes[?Required==`true`]'`
- Check app client settings: `aws cognito-idp describe-user-pool-client --user-pool-id <pool-id> --client-id <client-id>`
- Check if pre-sign-up trigger is configured: `aws cognito-idp describe-user-pool --user-pool-id <pool-id> --query 'UserPool.LambdaConfig.PreSignUp'`

SHOULD:
- Check pre-sign-up Lambda logs if trigger exists: `aws logs filter-log-events --log-group-name /aws/lambda/<pre-signup-function> --start-time <epoch-ms>`
- Check if username already exists: `aws cognito-idp admin-get-user --user-pool-id <pool-id> --username <username>`
- Check user pool email/SMS configuration: `aws cognito-idp describe-user-pool --user-pool-id <pool-id> --query 'UserPool.{EmailConfig:EmailConfiguration,SmsConfig:SmsConfiguration}'`

MAY:
- Check CloudWatch metrics for sign-up attempts: `aws cloudwatch get-metric-statistics --namespace AWS/Cognito --metric-name SignUpSuccesses --dimensions Name=UserPool,Value=<pool-id> --start-time <time> --end-time <time> --period 300 --statistics Sum`
- Check if the user pool is at the account limit

## Phase 2 — Remediate

MUST:
- Fix password policy if passwords are being rejected
- Add missing required attributes to the sign-up request
- Fix pre-sign-up Lambda if it is rejecting valid users
- Configure email/SMS delivery if verification is failing

SHOULD:
- Update app client to include all required OAuth scopes
- Ensure the app client allows the USER_SRP_AUTH or USER_PASSWORD_AUTH flow

MAY:
- Enable advanced security to monitor sign-up patterns

## Common Issues

- symptoms: "InvalidParameterException: Missing required attribute"
  diagnosis: "Sign-up request does not include all required user pool attributes."
  resolution: "Check required attributes with describe-user-pool and include them in the sign-up request."

- symptoms: "UsernameExistsException"
  diagnosis: "A user with this username or email already exists."
  resolution: "Check if the user exists. If using email as alias, another user may have the same email."

- symptoms: "InvalidPasswordException"
  diagnosis: "Password does not meet the user pool password policy."
  resolution: "Check password policy requirements (length, uppercase, lowercase, numbers, special characters)."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix password policy | YELLOW | State-changing but recoverable; policy can be reverted |
| Fix pre-sign-up Lambda | YELLOW | Lambda code change; previous version can be restored |
| Configure email/SMS delivery | YELLOW | Configuration change; recoverable |
| Enable advanced security | YELLOW | Adds monitoring; can be disabled |

## Escalation Conditions

- Production user pool password policy changes affecting all users
- Pre-sign-up Lambda trigger modifications in production
- Changes to required attributes on a user pool with existing users
- Email/SMS configuration changes affecting production delivery

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-user-pool` (password policy) | LOW | Pool configuration metadata |
| `describe-user-pool` (schema attributes) | LOW | Attribute schema, no user data |
| `describe-user-pool-client` | MEDIUM | Contains client configuration details |
| `admin-get-user` | HIGH | Returns user attributes and PII |
| `filter-log-events` (Lambda logs) | HIGH | May contain user data from sign-up attempts |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production without a rollback plan
- NEVER suggest disabling advanced security features
- NEVER expose or log user passwords or sensitive attributes
- NEVER delete a user pool to resolve configuration issues

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Password policy change | Revert to previous policy values via `update-user-pool` |
| Pre-sign-up Lambda fix | Redeploy previous Lambda version or alias |
| Email/SMS config change | Restore previous SES/SNS configuration |
| App client update | Revert app client settings via `update-user-pool-client` |

## Output Format

```yaml
root_cause: "sign_up_failure — <specific_cause>"
evidence:
  - type: user_pool_config
    content: "<password policy or required attributes>"
  - type: error_message
    content: "<exact error from application>"
severity: HIGH
mitigation:
  immediate: "Fix the sign-up request or user pool configuration"
  long_term: "Add client-side validation matching user pool policies"
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
