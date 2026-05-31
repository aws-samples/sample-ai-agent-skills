---
title: "H1 — Adaptive Authentication Issues"
description: "Diagnose adaptive authentication false positives and configuration issues"
status: active
severity: MEDIUM
triggers:
  - "adaptive auth"
  - "risk-based"
  - "blocked sign-in"
  - "advanced security"
  - "risk evaluation"
owner: devops-agent
objective: "Identify and fix adaptive authentication false positives or misconfigurations"
context: "Adaptive authentication evaluates sign-in risk based on device, location, and IP. It can block, require MFA, or allow sign-ins based on risk level. False positives block legitimate users. This feature requires advanced security mode (additional cost per MAU)."
---

## Phase 1 — Triage

MUST:
- Check advanced security configuration: `aws cognito-idp describe-user-pool --user-pool-id <pool-id> --query 'UserPool.UserPoolAddOns'`
- Check user auth events for risk decisions: `aws cognito-idp admin-list-user-auth-events --user-pool-id <pool-id> --username <username> --max-results 10`
- Verify the risk configuration: `aws cognito-idp describe-risk-configuration --user-pool-id <pool-id>`

SHOULD:
- Check if specific IPs are being blocked
- Review the risk level thresholds (low, medium, high)
- Check if the user's device/location changed recently

MAY:
- Check IP exception lists in the risk configuration
- Review overall risk evaluation patterns

## Phase 2 — Remediate

MUST:
- Add legitimate IPs to the exception list if being falsely blocked
- Adjust risk level actions (block → MFA → allow) based on requirements

SHOULD:
- Configure IP exception lists for known corporate IPs
- Set appropriate actions for each risk level

MAY:
- Implement custom challenge for medium-risk sign-ins instead of blocking

## Common Issues

- symptoms: "Legitimate users blocked from signing in"
  diagnosis: "Adaptive auth is set to block on medium or low risk, which is too aggressive."
  resolution: "Change medium-risk action to MFA challenge instead of block."

- symptoms: "All sign-ins from new location require MFA"
  diagnosis: "High-risk action is set to MFA and new locations are flagged as high risk."
  resolution: "Add known IP ranges to the exception list or adjust risk thresholds."

- symptoms: "Advanced security features not working"
  diagnosis: "UserPoolAddOns is not set to ENFORCED mode."
  resolution: "Enable advanced security in ENFORCED mode (AUDIT mode only logs, doesn't act)."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Add IPs to exception list | YELLOW | Risk config change; recoverable |
| Adjust risk level actions | YELLOW | Risk config change; recoverable |
| Change to MFA instead of block | YELLOW | Less restrictive; recoverable |

## Escalation Conditions

- Production user pool advanced security configuration changes
- Risk threshold changes affecting all users
- Changes to adaptive authentication enforcement mode

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-risk-configuration` | MEDIUM | Risk thresholds and IP exceptions |
| `admin-list-user-auth-events` | HIGH | Auth history with risk decisions and IPs |
| `describe-user-pool` (UserPoolAddOns) | LOW | Security mode configuration |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production
- NEVER suggest disabling advanced security features
- NEVER set all risk levels to ALLOW without explicit approval

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| IP exception list change | Remove IPs via `update-risk-configuration` |
| Risk level action change | Revert via `update-risk-configuration` |
| Enforcement mode change | Revert via `update-user-pool` (UserPoolAddOns) |

## Output Format

```yaml
root_cause: "adaptive_auth_issue — <specific_cause>"
evidence:
  - type: risk_config
    content: "<risk configuration>"
  - type: auth_events
    content: "<recent auth events with risk decisions>"
severity: MEDIUM
mitigation:
  immediate: "Adjust risk thresholds or add IP exceptions"
  long_term: "Monitor false positive rates and tune risk configuration"
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
