---
title: "H2 — Compromised Credentials Detection Issues"
description: "Diagnose compromised credentials detection configuration and response"
status: active
severity: HIGH
triggers:
  - "compromised credentials"
  - "credential breach"
  - "password leaked"
  - "CompromisedCredentialsRiskConfiguration"
owner: devops-agent
objective: "Identify and configure compromised credentials detection properly"
context: "Cognito can detect when users sign in with credentials known to be compromised (from public breaches). It can block sign-in or allow with logging. This requires advanced security mode (additional cost)."
---

## Phase 1 — Triage

MUST:
- Check advanced security mode: `aws cognito-idp describe-user-pool --user-pool-id <pool-id> --query 'UserPool.UserPoolAddOns'`
- Check compromised credentials configuration: `aws cognito-idp describe-risk-configuration --user-pool-id <pool-id>`
- Check user auth events: `aws cognito-idp admin-list-user-auth-events --user-pool-id <pool-id> --username <username> --max-results 10`

SHOULD:
- Verify which events trigger detection (SIGN_IN, SIGN_UP, PASSWORD_CHANGE)
- Check the action configured (BLOCK or ALLOW)
- Review recent compromised credential events in CloudWatch

MAY:
- Check if the user's password was recently changed
- Review the overall compromised credentials event rate

## Phase 2 — Remediate

MUST:
- If blocking legitimate users: change action from BLOCK to ALLOW (with logging)
- Force password reset for users with compromised credentials: `aws cognito-idp admin-reset-user-password --user-pool-id <pool-id> --username <username>`

SHOULD:
- Configure compromised credentials detection for SIGN_IN events at minimum
- Set up notifications for compromised credential detections

MAY:
- Implement a password policy that prevents reuse of compromised passwords

## Common Issues

- symptoms: "Users blocked with no clear reason"
  diagnosis: "Compromised credentials detection is blocking users whose passwords appear in breach databases."
  resolution: "Check auth events for CompromisedCredential risk. Force password reset for affected users."

- symptoms: "Compromised credentials detection not working"
  diagnosis: "Advanced security is in AUDIT mode (logs only) or not enabled."
  resolution: "Set UserPoolAddOns to ENFORCED mode."

- symptoms: "Too many users being flagged"
  diagnosis: "Many users are using passwords from known breaches."
  resolution: "Force password resets. Implement stronger password policies."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Change action from BLOCK to ALLOW | YELLOW | Risk config change; recoverable |
| `admin-reset-user-password` | YELLOW | Per-user change; user can re-authenticate |
| Configure detection events | YELLOW | Configuration change; recoverable |

## Escalation Conditions

- Production user pool advanced security configuration changes
- Compromised credentials detection mode changes
- Bulk password resets for affected users

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-risk-configuration` | MEDIUM | Compromised credentials settings |
| `admin-list-user-auth-events` | HIGH | Auth history with compromise indicators |
| `describe-user-pool` (UserPoolAddOns) | LOW | Security mode configuration |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production
- NEVER suggest disabling advanced security features
- NEVER disable compromised credentials detection without explicit approval

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Detection action change | Revert via `update-risk-configuration` |
| Password reset | User re-authenticates with new password; no rollback needed |
| Detection event config | Revert via `update-risk-configuration` |

## Output Format

```yaml
root_cause: "compromised_credentials_issue — <specific_cause>"
evidence:
  - type: security_config
    content: "<advanced security configuration>"
  - type: auth_events
    content: "<compromised credential events>"
severity: HIGH
mitigation:
  immediate: "Force password reset for affected users"
  long_term: "Enforce strong password policies and enable compromised credential blocking"
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
