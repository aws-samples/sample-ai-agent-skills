---
title: "C3 — Post-Confirmation Trigger Issues"
description: "Diagnose post-confirmation Lambda trigger failures"
status: active
severity: MEDIUM
triggers:
  - "post-confirmation"
  - "PostConfirmation"
  - "after sign-up"
  - "welcome email"
  - "user created trigger"
owner: devops-agent
objective: "Identify why the post-confirmation trigger is failing"
context: "The post-confirmation trigger runs after a user is confirmed. It is commonly used to add users to external systems, send welcome emails, or create database records. Failures do not block user confirmation but may leave downstream systems inconsistent."
---

## Phase 1 — Triage

MUST:
- Check trigger configuration: `aws cognito-idp describe-user-pool --user-pool-id <pool-id> --query 'UserPool.LambdaConfig.PostConfirmation'`
- Check Lambda logs: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch-ms>`
- Check Lambda errors and duration metrics
- Verify the function has permissions to access downstream services

SHOULD:
- Check if the trigger is failing silently (user confirmed but side effects missing)
- Verify Lambda execution role has required permissions (DynamoDB, SES, SNS, etc.)
- Check if the function is exceeding the 5-second timeout

MAY:
- Check downstream systems for missing records
- Test the Lambda function with a sample post-confirmation event

## Phase 2 — Remediate

MUST:
- Fix Lambda function errors
- Ensure downstream service permissions are correct
- Optimize to complete within 5 seconds

SHOULD:
- Move heavy operations to async processing (SQS, EventBridge)
- Add idempotency to handle retries

MAY:
- Implement a dead-letter queue for failed trigger invocations

## Common Issues

- symptoms: "User confirmed but not added to external database"
  diagnosis: "Post-confirmation Lambda is failing but user confirmation still succeeds."
  resolution: "Check Lambda logs. Fix the function. Consider async processing for reliability."

- symptoms: "Post-confirmation trigger timeout"
  diagnosis: "External service calls exceed the 5-second limit."
  resolution: "Send to SQS/EventBridge and process asynchronously."

- symptoms: "Duplicate records in downstream system"
  diagnosis: "Trigger may be invoked multiple times (Cognito retries on certain failures)."
  resolution: "Implement idempotency using the user sub as a unique key."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix Lambda function errors | YELLOW | Code change; previous version restorable |
| Fix downstream permissions | YELLOW | IAM change; recoverable |
| Move to async processing | YELLOW | Architecture change; reversible |

## Escalation Conditions

- Production user pool trigger changes
- Post-confirmation Lambda modifications affecting downstream systems

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-user-pool` (PostConfirmation) | LOW | Trigger ARN only |
| `filter-log-events` (Lambda logs) | HIGH | May contain user data |
| Lambda error/duration metrics | LOW | Operational metrics only |

## Prohibited Actions

- NEVER disable MFA to fix authentication issues
- NEVER suggest removing Cognito triggers in production without a rollback plan
- NEVER suggest disabling advanced security features

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Lambda code fix | Redeploy previous Lambda version or alias |
| Permission change | Revert IAM policy to previous version |
| Async migration | Revert to synchronous trigger processing |

## Output Format

```yaml
root_cause: "post_confirmation_trigger_failure — <specific_cause>"
evidence:
  - type: lambda_logs
    content: "<error logs>"
  - type: downstream_state
    content: "<state of downstream systems>"
severity: MEDIUM
mitigation:
  immediate: "Fix Lambda function or downstream permissions"
  long_term: "Move to async processing with dead-letter queue"
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
