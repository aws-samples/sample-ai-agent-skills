---
title: "H1 — Event Stream and Analytics Issues"
description: "Diagnose event stream configuration and analytics data problems"
status: active
severity: MEDIUM
triggers:
  - "events not streaming"
  - "analytics missing"
  - "Kinesis stream errors"
  - "event data not appearing"
owner: devops-agent
objective: "Identify and resolve event stream and analytics issues"
context: "Pinpoint can stream events to Kinesis Data Streams or Kinesis Data Firehose. Event streams require IAM role with write permissions. Missing permissions cause silent event loss."
---

## Phase 1 — Triage

MUST:
- Check event stream config: `aws pinpoint get-event-stream --application-id <app-id>`
- Verify Kinesis stream exists: `aws kinesis describe-stream --stream-name <stream-name>`
- Check IAM role permissions for the event stream role
- Verify Kinesis stream is ACTIVE

SHOULD:
- Check Kinesis stream metrics for incoming records
- Verify IAM role trust policy allows pinpoint.amazonaws.com
- Check for Kinesis throttling: `aws cloudwatch get-metric-statistics --namespace AWS/Kinesis --metric-name WriteProvisionedThroughputExceeded --dimensions Name=StreamName,Value=<stream> --start-time <start> --end-time <end> --period 300 --statistics Sum`

MAY:
- Check Kinesis Data Firehose delivery errors if using Firehose
- Review event types being streamed

## Phase 2 — Remediate

MUST:
- Fix IAM role with kinesis:PutRecords and kinesis:DescribeStream permissions
- Ensure trust policy allows pinpoint.amazonaws.com
- Verify Kinesis stream has sufficient shard capacity

SHOULD:
- Increase Kinesis shard count if throttling
- Set up CloudWatch alarms for stream health

MAY:
- Implement dead letter handling for failed events

## Common Issues

- symptoms: "Event stream configured but no data in Kinesis"
  diagnosis: "IAM role lacks permissions or trust policy incorrect."
  resolution: "Fix IAM role. Add kinesis:PutRecords permission. Update trust policy."

- symptoms: "Events intermittently missing"
  diagnosis: "Kinesis stream throttling due to insufficient shards."
  resolution: "Increase shard count. Monitor WriteProvisionedThroughputExceeded metric."

## Output Format

```yaml
root_cause: "event_stream — <specific_cause>"
evidence:
  - type: event_stream_config
    content: "<event stream configuration>"
  - type: kinesis_status
    content: "<Kinesis stream status and metrics>"
severity: MEDIUM
mitigation:
  immediate: "Fix IAM permissions or stream capacity"
  long_term: "Implement event stream monitoring and alerting"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — event streams contain customer interaction data
- Service Disruption Risk: LOW — stream issues do not affect message delivery
- Compliance Impact: HIGH — event data must comply with data protection regulations

## Escalation Conditions
- Event stream delivery completely stops
- Event data contains PII that should be filtered
- Stream destination becomes unavailable
- Event volume causes cost concerns
- Event data integrity issues detected

## Data Sensitivity
- Customer interaction events — RESTRICTED
- Event stream configurations — INTERNAL
- Destination configurations — INTERNAL
- Event metadata — INTERNAL
- Never expose customer event data in troubleshooting

## Prohibited Actions
- Do NOT stream events to unsecured destinations
- Do NOT include PII in event stream data without encryption
- Do NOT share event stream access with unauthorized parties
- Do NOT ignore event stream delivery failures
- Do NOT retain event data beyond approved retention periods
- Do NOT disable event streams without understanding downstream impact

## Phase 3 — Rollback
- Restore previous event stream configurations
- Re-enable disabled event streams
- Restore destination configurations
- Verify event delivery after changes
- Document event stream issues and data handling procedures
- Update event stream management procedures

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
