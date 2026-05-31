---
title: "C1 — Pre-Sign-Up Trigger Issues"
description: "Diagnose pre-sign-up Lambda trigger failures"
status: active
severity: MEDIUM
triggers:
  - "pre-sign-up"
  - "PreSignUp"
  - "sign-up trigger"
  - "auto-confirm"
  - "auto-verify"
owner: devops-agent
objective: "Identify why the pre-sign-up trigger is failing or behaving unexpectedly"
context: "The pre-sign-up trigger runs before Cognito creates the user. It can auto-confirm users, auto-verify attributes, or reject sign-ups. It must complete within 5 seconds and return the correct response format."
---

## Phase 1 — Triage

MUST:
- Check trigger configuration: `aws cognito-idp describe-user-pool --user-pool-id <pool-id> --query 'UserPool.LambdaConfig.PreSignUp'`
- Check Lambda function exists and is healthy: `aws lambda get-function --function-name <function-name>`
- Check Lambda logs for errors: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch-ms>`
- Check Lambda execution duration: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Duration --dimensions Name=FunctionName,Value=<function-name> --start-time <time> --end-time <time> --period 300 --statistics Maximum`

SHOULD:
- Verify Lambda has permission to be invoked by Cognito
- Check if the function is returning the correct response format
- Check Lambda error metrics

MAY:
- Test the Lambda function with a sample Cognito trigger event
- Check if cold starts are causing timeout issues

## Phase 2 — Remediate

MUST:
- Fix Lambda function errors or response format
- Ensure function completes within 5 seconds
- Verify Cognito has invoke permission on the Lambda function

SHOULD:
- Add proper error handling and logging
- Use provisioned concurrency if cold starts cause timeouts

MAY:
- Implement circuit breaker pattern for external service calls in the trigger

## Common Issues

- symptoms: "Sign-up succeeds but user is not auto-confirmed"
  diagnosis: "Pre-sign-up trigger is not setting autoConfirmUser to true in the response."
  resolution: "Return event.response.autoConfirmUser = true in the Lambda function."

- symptoms: "Sign-up fails with Lambda error"
  diagnosis: "Pre-sign-up Lambda is throwing an unhandled exception."
  resolution: "Check Lambda logs. Add try-catch and return proper error messages."

- symptoms: "Sign-up times out"
  diagnosis: "Pre-sign-up Lambda exceeds the 5-second Cognito timeout."
  resolution: "Optimize the function. Remove external API calls or make them faster."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix Lambda function errors | YELLOW | Code change; previous version restorable |
| Fix Cognito invoke permission | YELLOW | Permission change; recoverable |
| Use provisioned concurrency | YELLOW | Configuration change; recoverable |

## Escalation Conditions

- Production user pool trigger changes
- Lambda function modifications affecting sign-up flow

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-user-pool` (PreSignUp) | LOW | Trigger ARN only |
| `filter-log-events` (Lambda logs) | HIGH | May contain user sign-up data |
| Lambda duration/error metrics | LOW | Operational metrics only |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production without a rollback plan
- NEVER suggest disabling advanced security features

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Lambda code fix | Redeploy previous Lambda version or alias |
| Permission change | Revert Lambda resource policy |
| Provisioned concurrency | Remove provisioned concurrency setting |

## Output Format

```yaml
root_cause: "pre_sign_up_trigger_failure — <specific_cause>"
evidence:
  - type: lambda_logs
    content: "<error logs>"
  - type: execution_duration
    content: "<duration metrics>"
severity: MEDIUM
mitigation:
  immediate: "Fix Lambda function or remove trigger if not needed"
  long_term: "Add monitoring and alerting on trigger errors and duration"
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
