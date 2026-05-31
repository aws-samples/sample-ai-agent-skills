---
title: "D1 — Lambda Fulfillment Errors"
description: "Diagnose Lambda fulfillment function failures"
status: active
severity: HIGH
triggers:
  - "fulfillment failed"
  - "Lambda error in Lex"
  - "fulfillment timeout"
  - "Lambda not invoked"
owner: devops-agent
objective: "Identify and resolve Lambda fulfillment failures"
context: "Lambda fulfillment executes business logic when an intent is fulfilled. Lex enforces a 30-second timeout. Common failures: permission errors, timeout, response format errors, or function exceptions."
---

## Phase 1 — Triage

MUST:
- Check intent fulfillment config: `aws lexv2-models describe-intent --bot-id <bot-id> --bot-version DRAFT --locale-id en_US --intent-id <intent-id> --query 'fulfillmentCodeHook'`
- Check Lambda function exists: `aws lambda get-function --function-name <function-name>`
- Check Lambda resource policy allows Lex: `aws lambda get-policy --function-name <function-name>`
- Check Lambda logs: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch-ms> --limit 20`

SHOULD:
- Verify Lambda response format matches Lex V2 expected format
- Check Lambda timeout (must be ≤30 seconds for Lex)
- Test Lambda independently with sample Lex event

MAY:
- Check Lambda concurrency limits
- Review Lambda error metrics in CloudWatch

## Phase 2 — Remediate

MUST:
- Add Lex invoke permission to Lambda: `aws lambda add-permission --function-name <fn> --statement-id lex-invoke --action lambda:InvokeFunction --principal lexv2.amazonaws.com --source-arn <bot-arn>`
- Set Lambda timeout ≤30 seconds
- Return correct Lex V2 response format

SHOULD:
- Implement error handling in Lambda function
- Add CloudWatch alarms for Lambda errors

MAY:
- Use Lambda layers for shared dependencies

## Common Issues

- symptoms: "Fulfillment returns 'An error has occurred'"
  diagnosis: "Lambda function throwing unhandled exception."
  resolution: "Check Lambda logs. Fix exception. Add error handling."

- symptoms: "Fulfillment times out"
  diagnosis: "Lambda execution exceeds 30-second Lex limit."
  resolution: "Optimize Lambda. Use async pattern for long operations."

## Output Format

```yaml
root_cause: "lambda_fulfillment — <specific_cause>"
evidence:
  - type: lambda_logs
    content: "<Lambda execution logs>"
  - type: lambda_config
    content: "<Lambda configuration>"
severity: HIGH
mitigation:
  immediate: "Fix Lambda function or permissions"
  long_term: "Implement fulfillment monitoring and error handling"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete or overwrite a published bot alias that is actively serving production traffic
- **NEVER** modify intent or slot configurations on a live bot version without creating a new draft version first
- **NEVER** remove Lambda fulfillment hooks from intents without verifying downstream dependencies
- **NEVER** disable conversation logs on bots subject to compliance or audit requirements
- **NEVER** migrate a V1 bot to V2 in-place without maintaining the V1 bot as a fallback

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
