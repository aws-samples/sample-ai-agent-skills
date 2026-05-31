---
title: "F2 — Failed Event Analysis"
description: "Analyze failed events from DLQ to identify delivery failure root causes"
status: active
severity: MEDIUM
triggers:
  - "failed events"
  - "DLQ messages"
  - "analyze failures"
  - "delivery errors"
  - "event errors"
owner: devops-agent
objective: "Analyze failed events from DLQ to identify and resolve delivery failure patterns"
context: "DLQ messages contain the original event, error information, and metadata. Analyzing DLQ messages reveals delivery failure patterns including target errors, permission issues, payload problems, and throttling."
---

## Phase 1 — Triage

MUST:
- Receive messages from DLQ: `aws sqs receive-message --queue-url <dlq-url> --max-number-of-messages 10 --message-attribute-names All`
- Check message attributes for ERROR_CODE and ERROR_MESSAGE
- Identify the target that failed (RULE_ARN or pipe name)
- Check the original event payload for issues

SHOULD:
- Categorize failures by error type (permission, throttling, payload, target error)
- Check DLQ message count for failure volume: `aws sqs get-queue-attributes --queue-url <dlq-url> --attribute-names ApproximateNumberOfMessages`
- Correlate failures with target health metrics
- Review failure timestamps for patterns

MAY:
- Set up automated DLQ processing with Lambda
- Create failure analysis dashboards
- Implement event replay from DLQ after fixing issues

## Phase 2 — Remediate

MUST:
- Fix the identified root cause (permissions, target configuration, payload)
- Verify the fix by monitoring for new DLQ messages
- Process or purge analyzed DLQ messages

SHOULD:
- Implement automated DLQ monitoring and alerting
- Create runbooks for common failure patterns
- Set up DLQ processing automation

MAY:
- Implement event replay from DLQ
- Create failure analysis reports

## Common Issues

- symptoms: "DLQ messages with ACCESS_DENIED error"
  diagnosis: "Execution role lacks permission to invoke the target."
  resolution: "Add target-specific permissions to the execution role."

- symptoms: "DLQ messages with THROTTLING error"
  diagnosis: "Target service is throttling requests."
  resolution: "Increase target service limits. Implement backoff in retry policy."

- symptoms: "DLQ messages with PAYLOAD_TOO_LARGE error"
  diagnosis: "Event payload exceeds the target's maximum input size."
  resolution: "Reduce event payload size. Use input transformation to trim unnecessary fields."

## Output Format

```yaml
root_cause: "failed_event — <specific_cause>"
evidence:
  - type: dlq_messages
    content: "<DLQ message analysis with error codes>"
  - type: failure_pattern
    content: "<categorized failure types and counts>"
  - type: target
    content: "<target health and configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix root cause and process DLQ messages"
  long_term: "Implement DLQ monitoring and automated processing"
```


## Safety Ratings
```
safety_ratings:
  - "Receive messages from DLQ: GREEN — read-only message inspection (use receive, not delete)"
  - "Check message attributes for errors: GREEN — read-only analysis"
  - "Check DLQ message count: GREEN — read-only API call"
  - "Process or purge DLQ messages: YELLOW — removes messages from queue permanently"
  - "Fix target permissions: YELLOW — changes access scope"
  - "Implement DLQ processing automation: YELLOW — adds Lambda trigger to DLQ"
```

## Escalation Conditions
- DLQ contains events from production workflows requiring reprocessing
- Failure pattern indicates systemic target service issue
- ACCESS_DENIED errors requiring IAM policy changes
- THROTTLING errors requiring target service limit increases
- PAYLOAD_TOO_LARGE errors requiring event schema changes

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "DLQ message bodies: original event payloads with business data"
    - "Error codes and messages: target service failure details"
    - "Rule ARNs: infrastructure routing configuration"
  handling: "DLQ messages contain original event payloads. Handle as sensitive business data. Redact before sharing."
```

## Prohibited Actions
- NEVER purge DLQ messages before completing root cause analysis
- NEVER share DLQ message contents externally without redacting sensitive data
- NEVER replay DLQ events without fixing the underlying failure cause first
- NEVER delete the DLQ while it contains unanalyzed failure messages

## Phase 3 — Rollback
- If DLQ messages were purged: CANNOT be recovered — ensure analysis was completed first
- If target permissions were changed: revert IAM policy to previous version
- If DLQ processing Lambda was added: remove the Lambda trigger from the DLQ
- If events were replayed from DLQ: implement deduplication if targets received duplicates

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
