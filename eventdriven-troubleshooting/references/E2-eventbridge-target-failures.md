---
title: "E2 — EventBridge Target Failures"
description: "Diagnose and resolve Amazon EventBridge target invocation failures"
status: active
severity: HIGH
triggers:
  - "EventBridge target failed"
  - "target invocation error"
  - "FailedInvocations"
  - "EventBridge DLQ"
  - "target permission"
owner: devops-agent
objective: "Resolve EventBridge target invocation failures and ensure events are properly processed"
context: "EventBridge rules route matched events to targets (Lambda, SQS, SNS, Step Functions, ECS, etc.). Target invocations can fail due to permission issues, target unavailability, throttling, or input transformation errors. EventBridge retries failed invocations for up to 24 hours with exponential backoff. A DLQ can be configured to capture events that exhaust all retries."
---

## Phase 1 — Triage

MUST:
- Check FailedInvocations metric: `aws cloudwatch get-metric-statistics --namespace AWS/Events --metric-name FailedInvocations --dimensions Name=RuleName,Value=<rule-name> --start-time <time> --end-time <time> --period 300 --statistics Sum`
- List rule targets: `aws events list-targets-by-rule --rule <rule-name> --event-bus-name <bus-name>`
- Verify the EventBridge IAM role has permission to invoke the target: `aws iam get-role-policy --role-name <role-name> --policy-name <policy-name>`
- Check if the target resource exists and is accessible
- Check if a DLQ is configured and has messages: `aws events list-targets-by-rule --rule <rule-name>` (check DeadLetterConfig)

SHOULD:
- Check the target-specific error: Lambda throttling, SQS access denied, etc.
- Verify the input transformer configuration (if used): check InputTransformer in target config
- Review the EventBridge execution role trust policy
- Check CloudWatch Logs for target execution errors (Lambda logs, Step Functions execution history)

MAY:
- Check the retry policy configuration on the target
- Review if the target has resource-based policies that need updating
- Check if the event payload size exceeds the target's limits

## Phase 2 — Remediate

MUST:
- If permission denied: update the EventBridge IAM role to allow invoking the target
- If Lambda throttled: increase Lambda concurrency limit or reserved concurrency
- If SQS access denied: add EventBridge permission to the SQS queue policy
- If input transformer error: fix the input template syntax

SHOULD:
- Configure a DLQ for the target to capture failed events: `aws events put-targets --rule <rule-name> --targets '[{"Id":"<id>","Arn":"<target-arn>","DeadLetterConfig":{"Arn":"<dlq-arn>"}}]'`
- Adjust the retry policy for the target: `aws events put-targets --rule <rule-name> --targets '[{"Id":"<id>","Arn":"<target-arn>","RetryPolicy":{"MaximumRetryAttempts":3,"MaximumEventAgeInSeconds":3600}}]'`
- Verify target invocations succeed after remediation

MAY:
- Implement event replay from archive for missed events
- Set up CloudWatch alarms for FailedInvocations
- Document the target permissions and configuration

## Common Issues

- symptoms: "FailedInvocations increasing but no errors in target logs"
  diagnosis: "EventBridge cannot invoke the target — permission issue before the target is reached."
  resolution: "Check the EventBridge execution role. Add the required invoke/send permission for the target."

- symptoms: "Lambda target invoked but returns errors"
  diagnosis: "Lambda function errors are not FailedInvocations — they are Invocations with errors."
  resolution: "Check Lambda function logs. Fix the function code. EventBridge does not retry Lambda errors by default."

- symptoms: "Input transformer causes invocation failure"
  diagnosis: "Input template syntax error or referenced field not in the event."
  resolution: "Fix the input template. Use $.detail.fieldName syntax. Test with a sample event."

- symptoms: "Events lost — no DLQ configured"
  diagnosis: "Failed events exhausted retries (24 hours) with no DLQ to capture them."
  resolution: "Configure a DLQ for the target. Use event archive and replay for lost events."

## Output Format

```yaml
root_cause: "EventBridge target failure — <specific_cause>"
evidence:
  - type: failed_invocations
    content: "<FailedInvocations metric>"
  - type: target_config
    content: "<target configuration and permissions>"
severity: HIGH
mitigation:
  immediate: "Fix target permissions or configuration"
  long_term: "Configure DLQ, monitoring, and event replay capabilities"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (list/get/describe): GREEN — read-only"
  - "Phase 2 IAM role changes: YELLOW — affects EventBridge permissions"
  - "Phase 2 target configuration changes: YELLOW — affects event processing"

## Escalation Conditions

escalation_conditions:
  - "Critical events not reaching targets (alerts, workflows, data pipelines)"
  - "IAM role changes could affect other EventBridge rules"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Events have been lost and replay is needed"
  - "Issue involves cross-account target invocation"

## Data Sensitivity

data_sensitivity:
  - command: "list-targets-by-rule"
    sensitivity: MEDIUM
    contains: "Target ARNs and configuration"
  - command: "DLQ messages"
    sensitivity: HIGH
    contains: "Failed event payloads with business data"
  - command: "IAM role policies"
    sensitivity: HIGH
    contains: "Permission policies and resource ARNs"

## Prohibited Actions

prohibited_actions:
  - "NEVER grant EventBridge role overly broad permissions (e.g., * on all resources)"
  - "NEVER delete DLQ messages without analyzing the failures"
  - "NEVER remove targets from rules without verifying no events will be lost"
  - "NEVER modify EventBridge execution roles shared across multiple rules without review"

## Phase 3 — Rollback

- Before changing IAM roles, document the current policy
- If permission changes break other rules using the same role, revert the policy
- If target configuration changes cause failures, restore the previous target config
- If DLQ was added, verify it does not interfere with normal event processing
- Monitor FailedInvocations and target execution after any changes
