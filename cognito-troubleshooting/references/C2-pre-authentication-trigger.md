---
title: "C2 — Pre-Authentication Trigger Issues"
description: "Diagnose pre-authentication Lambda trigger failures"
status: active
severity: MEDIUM
triggers:
  - "pre-authentication"
  - "PreAuthentication"
  - "login trigger"
  - "auth trigger block"
owner: devops-agent
objective: "Identify why the pre-authentication trigger is failing or incorrectly blocking users"
context: "The pre-authentication trigger runs before Cognito authenticates the user. It can reject authentication attempts based on custom logic. It must complete within 5 seconds."
---

## Phase 1 — Triage

MUST:
- Check trigger configuration: `aws cognito-idp describe-user-pool --user-pool-id <pool-id> --query 'UserPool.LambdaConfig.PreAuthentication'`
- Check Lambda logs: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch-ms>`
- Check Lambda errors: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Errors --dimensions Name=FunctionName,Value=<function-name> --start-time <time> --end-time <time> --period 300 --statistics Sum`
- Verify the function is returning the event object correctly

SHOULD:
- Check if the trigger is throwing errors that block all sign-ins
- Verify Lambda execution role permissions
- Check execution duration against the 5-second limit

MAY:
- Test with a sample pre-authentication event payload
- Check if the trigger is making external calls that are timing out

## Phase 2 — Remediate

MUST:
- Fix Lambda function logic if it is incorrectly rejecting users
- Ensure the function returns the event object (not throwing for valid users)
- Fix timeout issues by optimizing code

SHOULD:
- Add proper logging to track which users are being rejected and why
- Implement graceful error handling

MAY:
- Add CloudWatch alarms for trigger error rates

## Common Issues

- symptoms: "All sign-ins fail after adding pre-authentication trigger"
  diagnosis: "Lambda function is throwing an error for all requests."
  resolution: "Check Lambda logs. Ensure the function returns the event object for allowed users."

- symptoms: "Intermittent sign-in failures"
  diagnosis: "Lambda cold starts or external service calls cause sporadic timeouts."
  resolution: "Optimize function. Use provisioned concurrency or cache external data."

- symptoms: "Specific users blocked unexpectedly"
  diagnosis: "Pre-authentication logic has incorrect filtering criteria."
  resolution: "Review the Lambda function logic and fix the user filtering conditions."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix Lambda function logic | YELLOW | Code change; previous version restorable |
| Temporarily remove trigger | RED | Removes security gate; affects all sign-ins |
| Fix timeout issues | YELLOW | Code optimization; recoverable |

## Escalation Conditions

- Production user pool trigger changes
- Pre-authentication Lambda modifications affecting all sign-ins
- Trigger removal consideration (even temporary)

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-user-pool` (PreAuthentication) | LOW | Trigger ARN only |
| `filter-log-events` (Lambda logs) | HIGH | May contain user auth data |
| Lambda error/duration metrics | LOW | Operational metrics only |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production without a rollback plan
- NEVER suggest disabling advanced security features

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Lambda code fix | Redeploy previous Lambda version or alias |
| Trigger removal | Re-add trigger via `update-user-pool` |

## Output Format

```yaml
root_cause: "pre_auth_trigger_failure — <specific_cause>"
evidence:
  - type: lambda_logs
    content: "<error or rejection logs>"
  - type: trigger_config
    content: "<trigger configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix Lambda function or temporarily remove trigger"
  long_term: "Add monitoring and fallback logic for trigger failures"
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
