---
title: "Z1 — General Cognito Troubleshooting (Catch-All)"
description: "Fallback SOP for Cognito issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown Cognito issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Identify the user pool: `aws cognito-idp describe-user-pool --user-pool-id <pool-id>`
- Check app client configuration: `aws cognito-idp describe-user-pool-client --user-pool-id <pool-id> --client-id <client-id>`
- Check Lambda triggers: `aws cognito-idp describe-user-pool --user-pool-id <pool-id> --query 'UserPool.LambdaConfig'`
- Check MFA configuration: `aws cognito-idp get-user-pool-mfa-config --user-pool-id <pool-id>`
- Check domain: `aws cognito-idp describe-user-pool --user-pool-id <pool-id> --query 'UserPool.Domain'`
- Check recent CloudWatch metrics for anomalies

SHOULD:
- Check identity providers: `aws cognito-idp list-identity-providers --user-pool-id <pool-id>`
- Check identity pool if applicable: `aws cognito-identity describe-identity-pool --identity-pool-id <identity-pool-id>`
- Review CloudTrail for recent Cognito API calls
- Check user status if a specific user is affected: `aws cognito-idp admin-get-user --user-pool-id <pool-id> --username <username>`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Sign-up/sign-in errors → Auth Flows (A1-A4)
- Federation failures → Federation (B1-B3)
- Lambda trigger issues → Triggers (C1-C3)
- MFA problems → MFA (D1-D2)
- Hosted UI errors → Hosted UI (E1-E2)
- App client issues → App Client (F1-F2)
- User management → User Management (G1-G2)
- Security features → Advanced (H1-H2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all user pool configurations systematically
- Review CloudTrail for recent configuration changes
- Check CloudWatch Logs for Lambda trigger errors
- Verify IAM permissions for the affected operations

SHOULD:
- Check AWS Health Dashboard for Cognito service events
- Compare with a known-good user pool configuration
- Test with a fresh user account

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Classify and route to specific SOP | GREEN | Read-only classification |
| Check all configurations | GREEN | Read-only investigation |
| Review CloudTrail events | GREEN | Read-only analysis |

## Escalation Conditions

- Production user pool changes of any kind
- Identity pool role mapping changes
- Unclassified issues after Phase 3 investigation
- Any issue affecting authentication for all users

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-user-pool` | MEDIUM | Pool configuration details |
| `describe-user-pool-client` | MEDIUM | Client configuration |
| `admin-get-user` | HIGH | User attributes and PII |
| CloudTrail events | MEDIUM | API call history |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production without a rollback plan
- NEVER suggest disabling advanced security features

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Route to specific SOP | Follow that SOP's rollback procedures |
| No direct remediation in catch-all | N/A — remediation happens in specific SOPs |

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<auth_flows|federation|triggers|mfa|hosted_ui|app_client|user_management|advanced|unknown>"
investigation_path: "user pool config → app client → triggers → metrics → <domain_classification>"
evidence:
  - type: user_pool_config
    content: "<configuration summary>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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
