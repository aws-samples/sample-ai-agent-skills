---
title: "D1 — SMS MFA Issues"
description: "Diagnose SMS-based multi-factor authentication failures"
status: active
severity: HIGH
triggers:
  - "SMS MFA"
  - "MFA code not received"
  - "SMS verification"
  - "phone verification"
  - "CodeMismatchException"
owner: devops-agent
objective: "Identify why SMS MFA codes are not being delivered or verified"
context: "SMS MFA uses Amazon SNS to deliver codes. Common failures include SNS sandbox restrictions, insufficient SNS spend limits, incorrect phone number format, and IAM role permissions for SNS."
---

## Phase 1 — Triage

MUST:
- Check MFA configuration: `aws cognito-idp get-user-pool-mfa-config --user-pool-id <pool-id>`
- Check SNS SMS settings: `aws sns get-sms-attributes`
- Check if SNS is in sandbox mode (sandbox only sends to verified numbers)
- Verify the user's phone number format: `aws cognito-idp admin-get-user --user-pool-id <pool-id> --username <username> --query 'UserAttributes[?Name==`phone_number`]'`

SHOULD:
- Check SNS delivery logs in CloudWatch
- Check SNS spend limit: `aws sns get-sms-attributes --attributes-names MonthlySpendLimit`
- Verify the IAM role for SNS: `aws cognito-idp describe-user-pool --user-pool-id <pool-id> --query 'UserPool.SmsConfiguration'`

MAY:
- Check SNS delivery statistics
- Test SMS delivery directly via SNS: `aws sns publish --phone-number <phone> --message "test"`

## Phase 2 — Remediate

MUST:
- Exit SNS sandbox for production: request via AWS Support
- Fix phone number format (must be E.164: +1234567890)
- Ensure the Cognito SNS IAM role has sns:Publish permission

SHOULD:
- Increase SNS monthly spend limit if needed
- Configure SNS delivery status logging

MAY:
- Set up an SNS origination identity for better deliverability
- Consider TOTP MFA as a fallback

## Common Issues

- symptoms: "MFA code never arrives"
  diagnosis: "SNS is in sandbox mode and the phone number is not verified."
  resolution: "Exit SNS sandbox for production or verify the phone number in sandbox."

- symptoms: "CodeMismatchException on MFA verification"
  diagnosis: "User is entering an expired or incorrect code."
  resolution: "Codes expire after 3 minutes. Ensure the user enters the most recent code."

- symptoms: "SMS delivery works for some numbers but not others"
  diagnosis: "Carrier filtering or country restrictions."
  resolution: "Check SNS delivery logs. Some countries require origination IDs."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Exit SNS sandbox | YELLOW | Account-level change; recoverable |
| Fix phone number format | GREEN | Data correction; read-only validation |
| Fix IAM role for SNS | YELLOW | Permission change; recoverable |
| Increase SNS spend limit | YELLOW | Limit change; recoverable |

## Escalation Conditions

- Production user pool MFA configuration changes
- SNS spend limit increases
- Changes affecting SMS delivery for all users

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-user-pool-mfa-config` | LOW | Pool-level MFA settings |
| `get-sms-attributes` | MEDIUM | SNS SMS configuration |
| `admin-get-user` (phone_number) | HIGH | User PII |
| SNS delivery logs | HIGH | Phone numbers and delivery status |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production
- NEVER suggest disabling advanced security features
- NEVER log or expose user phone numbers

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| SNS sandbox exit | Cannot re-enter sandbox; document the change |
| IAM role change | Revert IAM policy to previous version |
| SNS spend limit increase | Reduce limit via AWS Support |

## Output Format

```yaml
root_cause: "sms_mfa_failure — <specific_cause>"
evidence:
  - type: mfa_config
    content: "<MFA configuration>"
  - type: sns_settings
    content: "<SNS SMS attributes>"
severity: HIGH
mitigation:
  immediate: "Fix SNS configuration or phone number format"
  long_term: "Exit SNS sandbox, configure delivery logging, consider TOTP fallback"
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
