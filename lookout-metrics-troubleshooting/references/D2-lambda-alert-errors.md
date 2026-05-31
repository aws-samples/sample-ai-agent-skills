---
title: "D2 — Lambda Alert Errors"
description: "Diagnose and resolve Lambda function alert invocation failures"
status: active
severity: HIGH
triggers: ["Lambda alert not triggering", "Lambda invocation failed", "Lambda permission denied", "alert Lambda error"]
owner: devops-agent
objective: "Ensure Lambda function alerts are invoked successfully when anomalies are detected"
context: "Lambda alerts require both IAM role permissions AND a resource-based policy on the Lambda function. The most common failure is a missing resource-based policy granting Lookout for Metrics invoke permission."
---

## Phase 1 — Triage

MUST:
- Describe the alert to get Lambda function ARN: `aws lookoutmetrics describe-alert --alert-arn <arn>`
- Check Lambda function exists: `aws lambda get-function --function-name <name>`
- Check Lambda resource-based policy: `aws lambda get-policy --function-name <name>`

SHOULD:
- Check Lambda function logs for invocation errors: `aws logs filter-log-events --log-group-name /aws/lambda/<name> --start-time <epoch_ms>`
- Verify the Lambda function is not throttled: `aws lambda get-function-concurrency --function-name <name>`
- Check CloudTrail for InvokeFunction errors from Lookout for Metrics

MAY:
- Test Lambda function with a sample anomaly payload
- Review Lambda function timeout and memory configuration

## Phase 2 — Remediate

MUST:
- Add resource-based policy granting Lookout for Metrics invoke permission:
  ```
  aws lambda add-permission --function-name <name> --statement-id lookoutmetrics-invoke --action lambda:InvokeFunction --principal lookoutmetrics.amazonaws.com --source-arn <detector-arn>
  ```
- Fix Lambda function errors visible in CloudWatch Logs

SHOULD:
- Ensure Lambda function timeout is sufficient for the alert processing logic
- Verify Lambda function has appropriate IAM execution role for downstream actions
- Check Lambda function is in the correct region (same as detector)

MAY:
- Implement dead-letter queue for failed Lambda invocations
- Add CloudWatch alarms on Lambda error metrics

## Safety Ratings

```
safety_ratings:
  - "lambda get-function: GREEN — read-only function details"
  - "lambda get-policy: GREEN — read-only policy check"
  - "lambda add-permission: YELLOW — modifies function resource policy"
  - "logs filter-log-events: GREEN — read-only log access"
```

## Escalation Conditions

- Lambda function is in a different account and cross-account invocation cannot be configured
- Lambda function consistently times out processing anomaly payloads
- Resource-based policy is at the 20KB limit and cannot add more statements
- Lambda function errors indicate downstream service failures (e.g., cannot write to database)

## Data Sensitivity

- `aws lambda get-function`: MEDIUM — reveals function configuration and environment variables
- `aws lambda get-policy`: LOW — permission policy only
- `aws logs filter-log-events`: HIGH — may contain anomaly data and business metric values
- Lambda environment variables: HIGH — may contain secrets and connection strings

## Prohibited Actions

- NEVER grant `lambda:*` permissions to the Lookout for Metrics service principal
- NEVER expose Lambda environment variables containing secrets
- NEVER modify a production Lambda function's code to fix alert issues without testing
- NEVER remove existing resource-based policy statements when adding new ones

## Phase 3 — Rollback

1. If resource-based policy was added incorrectly, remove it: `aws lambda remove-permission --function-name <name> --statement-id lookoutmetrics-invoke`
2. If Lambda function code was modified, deploy the previous version
3. If Lambda concurrency was changed, revert to previous setting
4. Document all Lambda configuration changes
5. Verify alert delivery by monitoring the next anomaly detection cycle

## Common Issues

- symptoms: "Alert configured but Lambda never invoked"
  diagnosis: "Missing resource-based policy on the Lambda function for Lookout for Metrics."
  resolution: "Add permission: `aws lambda add-permission --function-name <name> --statement-id lookoutmetrics-invoke --action lambda:InvokeFunction --principal lookoutmetrics.amazonaws.com`"

- symptoms: "Lambda invoked but returns errors"
  diagnosis: "Lambda function code cannot parse the Lookout for Metrics anomaly payload."
  resolution: "Review Lambda logs. Update function code to handle the Lookout for Metrics event schema correctly."

## Output Format

```yaml
root_cause: "lambda_alert — <specific_cause>"
evidence:
  - type: lambda_error
    content: "<Lambda policy, logs, and error details>"
severity: HIGH
mitigation:
  immediate: "Fix Lambda permissions or function errors"
  long_term: "Implement Lambda error monitoring and dead-letter queue"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
