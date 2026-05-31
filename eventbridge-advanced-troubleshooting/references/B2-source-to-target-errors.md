---
title: "B2 — Source-to-Target Delivery Errors"
description: "Diagnose event delivery failures between pipe source and target"
status: active
severity: HIGH
triggers:
  - "pipe delivery failed"
  - "target error"
  - "source error"
  - "pipe not delivering"
  - "events not reaching target"
owner: devops-agent
objective: "Identify and resolve source-to-target delivery errors in EventBridge Pipes"
context: "Pipe delivery errors can occur at the source (read failures), enrichment (processing failures), or target (write failures). Each stage has different error handling. The pipe execution role must have permissions for all stages."
---

## Phase 1 — Triage

MUST:
- Describe the pipe: `aws pipes describe-pipe --name <pipe-name>`
- Check pipe CurrentState (RUNNING, STOPPED, CREATE_FAILED, UPDATE_FAILED)
- Check CloudWatch metrics: `aws cloudwatch get-metric-statistics --namespace AWS/EventBridge/Pipes --metric-name ExecutionFailed --dimensions Name=PipeName,Value=<pipe-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify IAM role has source read and target write permissions

SHOULD:
- Check DLQ for failed events: `aws sqs receive-message --queue-url <dlq-url> --max-number-of-messages 10`
- Verify source has events to process
- Check target health and accessibility
- Review enrichment function logs if enrichment is configured

MAY:
- Check source-specific metrics (SQS message count, Kinesis iterator age)
- Verify target-specific limits (Lambda concurrency, SQS message size)
- Review pipe execution logs

## Phase 2 — Remediate

MUST:
- Fix IAM role permissions for the failing stage (source, enrichment, or target)
- Ensure target is healthy and can accept events
- Restart the pipe if in a failed state: `aws pipes start-pipe --name <pipe-name>`

SHOULD:
- Configure DLQ to capture failed events for analysis
- Set appropriate retry policy
- Monitor pipe execution metrics

MAY:
- Implement alerting for pipe failures
- Add enrichment error handling

## Common Issues

- symptoms: "Pipe running but no events delivered to target"
  diagnosis: "Source has no events, filter is too restrictive, or target permissions are missing."
  resolution: "Verify source has events. Check filter pattern. Add target write permissions to the role."

- symptoms: "Events delivered but target returns errors"
  diagnosis: "Target rejects events due to payload format, size limits, or throttling."
  resolution: "Check target error messages in DLQ. Verify payload format matches target expectations."

- symptoms: "Pipe stops processing after errors"
  diagnosis: "Pipe encountered too many consecutive errors and stopped."
  resolution: "Fix the root cause. Restart the pipe with aws pipes start-pipe."

## Output Format

```yaml
root_cause: "source_to_target — <specific_cause>"
evidence:
  - type: pipe
    content: "<pipe status and configuration>"
  - type: metrics
    content: "<execution success/failure metrics>"
  - type: dlq
    content: "<failed events from DLQ>"
severity: HIGH
mitigation:
  immediate: "Fix permissions or target issues and restart pipe"
  long_term: "Configure DLQ, retry policy, and monitoring"
```


## Safety Ratings
```
safety_ratings:
  - "Describe pipe status: GREEN — read-only API call"
  - "Check CloudWatch execution metrics: GREEN — read-only monitoring query"
  - "Check DLQ for failed events: GREEN — read-only message inspection"
  - "Restart stopped pipe: YELLOW — resumes event processing, may process backlog"
  - "Fix IAM role permissions: YELLOW — changes access scope for the pipe"
  - "Modify pipe filter or enrichment: YELLOW — changes event processing behavior"
```

## Escalation Conditions
- Pipe serves production event processing pipeline
- Pipe in FAILED state with events accumulating in source
- Target service returning persistent errors
- DLQ filling up with failed events requiring analysis
- Fix requires coordination between source and target service teams

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "DLQ messages: failed event payloads with business data"
    - "Error details: target service state and error messages"
    - "Pipe execution metrics: processing volume and failure rates"
  handling: "DLQ messages may contain sensitive business data. Do not expose error details externally."
```

## Prohibited Actions
- NEVER purge DLQ messages before analyzing failure root cause
- NEVER restart a pipe without fixing the underlying delivery issue first
- NEVER modify pipe configuration while events are actively being processed without understanding impact
- NEVER delete a pipe's DLQ while it contains unprocessed failure messages

## Phase 3 — Rollback
- If pipe was restarted: stop the pipe if it continues to fail and events are being lost
- If IAM permissions were changed: revert to previous policy version
- If pipe filter was modified: restore previous filter criteria
- If enrichment configuration was changed: revert to previous enrichment settings
- If pipe was stopped for investigation: restart the pipe after root cause is resolved

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
