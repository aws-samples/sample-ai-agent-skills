---
title: "G1 — User Migration Trigger Issues"
description: "Diagnose user migration Lambda trigger failures"
status: active
severity: MEDIUM
triggers:
  - "user migration"
  - "migrate user"
  - "UserMigration"
  - "legacy user"
owner: devops-agent
objective: "Identify why user migration trigger is failing during sign-in"
context: "The user migration trigger runs when a user signs in but doesn't exist in the user pool. It allows migrating users from a legacy system on-demand. The trigger must return user attributes and must complete within 5 seconds."
---

## Phase 1 — Triage

MUST:
- Check trigger configuration: `aws cognito-idp describe-user-pool --user-pool-id <pool-id> --query 'UserPool.LambdaConfig.UserMigration'`
- Check Lambda logs: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch-ms>`
- Verify the function returns the correct response format with user attributes
- Check if USER_PASSWORD_AUTH is enabled (required for migration trigger)

SHOULD:
- Verify the Lambda can connect to the legacy user store
- Check if the function is timing out (5-second limit)
- Verify returned attributes match the user pool schema

MAY:
- Test the Lambda with a sample migration event
- Check if the legacy system is rate-limiting requests

## Phase 2 — Remediate

MUST:
- Enable USER_PASSWORD_AUTH on the app client (migration trigger only works with this flow)
- Fix Lambda function to return correct user attributes
- Ensure the function completes within 5 seconds

SHOULD:
- Add error handling for legacy system connectivity issues
- Log migration successes and failures for tracking

MAY:
- Consider bulk import as an alternative for large user bases

## Common Issues

- symptoms: "Migration trigger not invoked"
  diagnosis: "USER_PASSWORD_AUTH is not enabled on the app client. Migration only works with password-based auth."
  resolution: "Enable USER_PASSWORD_AUTH on the app client."

- symptoms: "User migrated but attributes missing"
  diagnosis: "Lambda function not returning all required attributes."
  resolution: "Return all required attributes in event.response.userAttributes."

- symptoms: "Migration trigger timeout"
  diagnosis: "Legacy system lookup exceeds 5 seconds."
  resolution: "Optimize legacy system queries. Consider caching or connection pooling."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable USER_PASSWORD_AUTH | YELLOW | App client change; recoverable |
| Fix Lambda function | YELLOW | Code change; previous version restorable |
| Optimize Lambda performance | YELLOW | Code change; recoverable |

## Escalation Conditions

- Production user pool trigger changes
- App client auth flow changes affecting active users
- Legacy system connectivity changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-user-pool` (UserMigration) | LOW | Trigger ARN only |
| `filter-log-events` (Lambda logs) | HIGH | May contain user credentials from migration |
| Lambda duration/error metrics | LOW | Operational metrics only |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production without a rollback plan
- NEVER suggest disabling advanced security features
- NEVER log user passwords during migration

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| USER_PASSWORD_AUTH enablement | Remove flow via `update-user-pool-client` |
| Lambda code fix | Redeploy previous Lambda version |
| Migration trigger removal | Re-add trigger via `update-user-pool` |

## Output Format

```yaml
root_cause: "user_migration_failure — <specific_cause>"
evidence:
  - type: trigger_config
    content: "<migration trigger configuration>"
  - type: lambda_logs
    content: "<error logs>"
severity: MEDIUM
mitigation:
  immediate: "Fix Lambda function or enable USER_PASSWORD_AUTH"
  long_term: "Complete bulk migration to eliminate dependency on legacy system"
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
