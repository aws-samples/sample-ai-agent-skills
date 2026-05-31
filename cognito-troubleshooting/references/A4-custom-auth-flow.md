---
title: "A4 — Cognito Custom Auth Flow Failures"
description: "Diagnose custom authentication flow issues requiring Lambda triggers"
status: active
severity: HIGH
triggers:
  - "custom auth"
  - "CUSTOM_AUTH"
  - "challenge"
  - "passwordless"
  - "OTP authentication"
owner: devops-agent
objective: "Identify why custom authentication flow is failing and fix trigger configuration"
context: "Custom auth flows require three Lambda triggers: Define Auth Challenge, Create Auth Challenge, and Verify Auth Challenge Response. Missing any trigger, incorrect challenge responses, or trigger timeouts cause failures. All triggers must complete within 5 seconds."
---

## Phase 1 — Triage

MUST:
- Verify all three triggers are configured: `aws cognito-idp describe-user-pool --user-pool-id <pool-id> --query 'UserPool.LambdaConfig.{Define:DefineAuthChallenge,Create:CreateAuthChallenge,Verify:VerifyAuthChallengeResponse}'`
- Check that CUSTOM_AUTH flow is enabled on the app client: `aws cognito-idp describe-user-pool-client --user-pool-id <pool-id> --client-id <client-id> --query 'UserPoolClient.ExplicitAuthFlows'`
- Check Lambda function logs for each trigger: `aws logs filter-log-events --log-group-name /aws/lambda/<define-auth-function> --start-time <epoch-ms>`
- Verify Lambda functions exist and are not erroring: `aws lambda get-function --function-name <function-name>`

SHOULD:
- Check Lambda execution duration (must be < 5 seconds): `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Duration --dimensions Name=FunctionName,Value=<function-name> --start-time <time> --end-time <time> --period 300 --statistics Maximum`
- Check Lambda error rate: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Errors --dimensions Name=FunctionName,Value=<function-name> --start-time <time> --end-time <time> --period 300 --statistics Sum`
- Verify Lambda execution role has required permissions

MAY:
- Test triggers independently with test events
- Check if the Define Auth Challenge trigger is returning the correct challenge name

## Phase 2 — Remediate

MUST:
- Add any missing Lambda triggers to the user pool
- Enable CUSTOM_AUTH flow on the app client
- Fix Lambda function errors or timeouts

SHOULD:
- Optimize trigger code to complete within 5 seconds
- Add proper error handling and logging to all three triggers
- Ensure Define Auth Challenge returns correct issueTokens and failAuthentication flags

MAY:
- Implement retry logic in the client application for transient failures

## Common Issues

- symptoms: "InvalidLambdaResponseException"
  diagnosis: "Lambda trigger returned an invalid response format."
  resolution: "Check the Lambda response matches the expected Cognito trigger response format."

- symptoms: "Custom auth flow hangs or times out"
  diagnosis: "One of the Lambda triggers is exceeding the 5-second timeout."
  resolution: "Optimize trigger code. Move heavy operations to async processing."

- symptoms: "Challenge loop — keeps asking for challenges"
  diagnosis: "Define Auth Challenge trigger never sets issueTokens to true."
  resolution: "Fix the Define Auth Challenge logic to set issueTokens=true after successful verification."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Add missing Lambda triggers | YELLOW | Configuration change; triggers can be removed |
| Enable CUSTOM_AUTH flow | YELLOW | App client change; recoverable |
| Fix Lambda function errors | YELLOW | Code change; previous version restorable |

## Escalation Conditions

- Production user pool Lambda trigger changes
- Custom auth flow modifications affecting active users
- Identity pool role mapping changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-user-pool` (LambdaConfig) | LOW | Trigger ARN configuration |
| `describe-user-pool-client` | MEDIUM | Client configuration details |
| `filter-log-events` (Lambda logs) | HIGH | May contain user auth data |
| Lambda duration/error metrics | LOW | Operational metrics only |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production without a rollback plan
- NEVER suggest disabling advanced security features
- NEVER hardcode secrets or credentials in Lambda trigger code

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Lambda trigger addition | Remove trigger via `update-user-pool` |
| CUSTOM_AUTH flow enablement | Remove flow via `update-user-pool-client` |
| Lambda code fix | Redeploy previous Lambda version or alias |

## Output Format

```yaml
root_cause: "custom_auth_failure — <specific_cause>"
evidence:
  - type: trigger_config
    content: "<configured triggers>"
  - type: lambda_logs
    content: "<relevant error logs>"
severity: HIGH
mitigation:
  immediate: "Fix missing triggers or Lambda errors"
  long_term: "Add comprehensive logging and monitoring to all auth triggers"
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
