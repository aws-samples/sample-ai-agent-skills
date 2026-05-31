---
title: "D2 — TOTP MFA Issues"
description: "Diagnose time-based one-time password MFA failures"
status: active
severity: HIGH
triggers:
  - "TOTP MFA"
  - "authenticator app"
  - "MFA code invalid"
  - "SOFTWARE_TOKEN_MFA"
  - "TOTP setup"
owner: devops-agent
objective: "Identify why TOTP MFA setup or verification is failing"
context: "TOTP MFA uses authenticator apps (Google Authenticator, Authy, etc.) to generate time-based codes. Failures are typically caused by clock skew, incorrect setup flow, or MFA not being enabled on the user pool."
---

## Phase 1 — Triage

MUST:
- Check MFA configuration: `aws cognito-idp get-user-pool-mfa-config --user-pool-id <pool-id>`
- Check if SOFTWARE_TOKEN_MFA is enabled
- Check user's MFA preferences: `aws cognito-idp admin-get-user --user-pool-id <pool-id> --username <username>`
- Verify the app client supports MFA challenge responses

SHOULD:
- Check if the user has completed TOTP setup (associated a software token)
- Verify the user's device clock is synchronized
- Check auth events for MFA failures: `aws cognito-idp admin-list-user-auth-events --user-pool-id <pool-id> --username <username> --max-results 10`

MAY:
- Check if the user has multiple MFA methods configured
- Verify the TOTP secret was correctly scanned/entered

## Phase 2 — Remediate

MUST:
- Enable SOFTWARE_TOKEN_MFA on the user pool if not enabled
- Guide user through TOTP setup if not completed
- Verify device clock synchronization

SHOULD:
- Reset MFA for the user if the TOTP secret is lost: `aws cognito-idp admin-set-user-mfa-preference --user-pool-id <pool-id> --username <username> --software-token-mfa-settings Enabled=false,PreferredMfa=false`
- Re-associate the software token

MAY:
- Offer SMS MFA as a fallback during TOTP issues

## Common Issues

- symptoms: "TOTP code always invalid"
  diagnosis: "Device clock is out of sync. TOTP codes are time-sensitive (30-second window)."
  resolution: "Sync the device clock. TOTP allows a small window of tolerance but significant drift causes failures."

- symptoms: "Cannot set up TOTP — no QR code"
  diagnosis: "The AssociateSoftwareToken API was not called or the secret was not presented to the user."
  resolution: "Call AssociateSoftwareToken, present the secret as a QR code, then call VerifySoftwareToken."

- symptoms: "MFA challenge not presented during sign-in"
  diagnosis: "MFA is set to OPTIONAL and the user hasn't enabled it, or the app client doesn't handle MFA challenges."
  resolution: "Set MFA to REQUIRED or ensure the user has opted in. Handle SOFTWARE_TOKEN_MFA challenge in the app."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable SOFTWARE_TOKEN_MFA | YELLOW | Pool-level config change; recoverable |
| Reset user MFA preference | YELLOW | Per-user change; recoverable |
| Re-associate software token | YELLOW | Per-user change; recoverable |

## Escalation Conditions

- Production user pool MFA configuration changes (OPTIONAL → REQUIRED)
- Bulk MFA resets for multiple users
- Changes affecting MFA enforcement for all users

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-user-pool-mfa-config` | LOW | Pool-level MFA settings |
| `admin-get-user` | HIGH | User attributes and MFA status |
| `admin-list-user-auth-events` | HIGH | Auth history with IP addresses |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production
- NEVER suggest disabling advanced security features
- NEVER expose TOTP secrets in logs or output

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| MFA pool config change | Revert via `set-user-pool-mfa-config` |
| User MFA preference reset | Re-enable via `admin-set-user-mfa-preference` |
| Software token re-association | Previous token is invalidated; user must re-setup |

## Output Format

```yaml
root_cause: "totp_mfa_failure — <specific_cause>"
evidence:
  - type: mfa_config
    content: "<user pool MFA settings>"
  - type: user_mfa_status
    content: "<user MFA preferences>"
severity: HIGH
mitigation:
  immediate: "Fix MFA configuration or reset user's TOTP"
  long_term: "Implement MFA recovery flow and backup codes"
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
