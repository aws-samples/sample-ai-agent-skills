---
title: "F1 — DLQ Configuration Issues"
description: "Diagnose dead letter queue configuration for EventBridge targets and Pipes"
status: active
severity: MEDIUM
triggers:
  - "DLQ configuration"
  - "dead letter queue"
  - "DLQ not receiving"
  - "failed events lost"
  - "DLQ setup"
owner: devops-agent
objective: "Identify and resolve DLQ configuration issues for EventBridge"
context: "DLQs capture events that fail delivery after all retries. DLQs are configured per-target (rules) or per-pipe. The execution role must have sqs:SendMessage permission on the DLQ. Without a DLQ, failed events are lost."
---

## Phase 1 — Triage

MUST:
- Check if DLQ is configured on the target/pipe
- Verify the DLQ SQS queue exists: `aws sqs get-queue-attributes --queue-url <dlq-url> --attribute-names All`
- Check IAM role has sqs:SendMessage on the DLQ
- Verify the DLQ is in the same region as the event bus/pipe

SHOULD:
- Check DLQ message count: `aws sqs get-queue-attributes --queue-url <dlq-url> --attribute-names ApproximateNumberOfMessages`
- Verify retry policy configuration on the target
- Check for DLQ access policy restrictions
- Review CloudTrail for DLQ-related errors

MAY:
- Check DLQ encryption configuration
- Verify DLQ retention period
- Monitor DLQ depth over time

## Phase 2 — Remediate

MUST:
- Create the DLQ SQS queue if it doesn't exist
- Configure the DLQ on the target or pipe
- Add sqs:SendMessage permission to the execution role for the DLQ ARN

SHOULD:
- Set appropriate retry policy (number of retries, max age)
- Configure DLQ retention period for failed event analysis
- Set up CloudWatch alarms for DLQ message count

MAY:
- Implement DLQ processing automation (Lambda trigger)
- Set up DLQ monitoring dashboards

## Common Issues

- symptoms: "DLQ configured but no messages received despite target failures"
  diagnosis: "IAM role lacks sqs:SendMessage permission on the DLQ."
  resolution: "Add sqs:SendMessage for the DLQ ARN to the execution role."

- symptoms: "Failed events lost — no DLQ configured"
  diagnosis: "Target has no DLQ configured. Failed events after retries are discarded."
  resolution: "Configure an SQS queue as DLQ on the target. Add IAM permissions."

- symptoms: "DLQ messages but cannot determine failure reason"
  diagnosis: "DLQ messages include error information in message attributes."
  resolution: "Check SQS message attributes for ERROR_CODE, ERROR_MESSAGE, and RULE_ARN."

## Output Format

```yaml
root_cause: "dlq_configuration — <specific_cause>"
evidence:
  - type: dlq
    content: "<DLQ queue configuration and message count>"
  - type: target
    content: "<target DLQ and retry configuration>"
  - type: iam_role
    content: "<execution role DLQ permissions>"
severity: MEDIUM
mitigation:
  immediate: "Configure DLQ and fix permissions"
  long_term: "Implement DLQ monitoring and processing automation"
```


## Safety Ratings
```
safety_ratings:
  - "Check DLQ queue attributes: GREEN — read-only API call"
  - "Check DLQ message count: GREEN — read-only API call"
  - "Verify IAM role DLQ permissions: GREEN — read-only IAM inspection"
  - "Create DLQ SQS queue: GREEN — creates new resource, no impact on existing"
  - "Configure DLQ on target: YELLOW — changes error handling behavior"
  - "Add sqs:SendMessage permission: YELLOW — grants DLQ write access to execution role"
```

## Escalation Conditions
- Failed events being lost due to missing DLQ configuration
- DLQ filling up indicating persistent target failures
- DLQ permissions preventing failure capture
- DLQ retention period too short causing loss of failure evidence

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "DLQ messages: failed event payloads with business data"
    - "Error codes and messages: target service failure details"
    - "DLQ queue URL: infrastructure resource identifier"
  handling: "DLQ messages contain failed event payloads that may include sensitive business data. Restrict queue access."
```

## Prohibited Actions
- NEVER purge a DLQ before analyzing the failure messages
- NEVER set DLQ retention period shorter than the investigation SLA
- NEVER share DLQ messages externally without redacting sensitive payload data
- NEVER remove DLQ configuration from production targets without an alternative error handling strategy

## Phase 3 — Rollback
- If DLQ was configured on a target: remove DLQ configuration (failed events will be lost going forward)
- If DLQ SQS queue was created: delete the queue if not needed (messages in queue will be lost)
- If IAM permissions were added: remove sqs:SendMessage permission from the execution role
- If retry policy was changed: restore previous retry configuration

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
