---
title: "C2 — Sink Connector Errors"
description: "Diagnose Managed Flink sink connector issues"
status: active
severity: HIGH
triggers: ["sink error", "cannot write to Kinesis", "S3 sink error", "output error", "destination error"]
owner: devops-agent
objective: "Resolve sink connector issues including Kinesis, S3, Kafka, and Firehose sink errors"
context: "Sink connectors write processed data to destinations. Errors include permissions, serialization, capacity, and connectivity issues."
---

## Phase 1 — Triage

MUST:
- Check application logs for sink errors: `aws logs filter-log-events --log-group-name /aws/kinesis-analytics/<app-name> --filter-pattern "sink" --start-time <epoch-ms> --limit 50`
- Verify destination exists and is accessible
- Check IAM role permissions for destination access

SHOULD:
- Check destination capacity (Kinesis shard count, S3 bucket, Firehose delivery stream)
- Verify serialization format matches destination expectations

MAY:
- Check destination throttling metrics
- Test destination connectivity independently

## Phase 2 — Remediate

MUST:
- Fix IAM role permissions for destination write access
- Fix serialization errors (match output format to destination schema)
- Ensure destination has sufficient capacity for the output volume

SHOULD:
- Configure sink buffering and batching for optimal throughput
- Implement retry logic for transient destination errors

MAY:
- Use exactly-once sink connectors where available (Kinesis, Kafka)
- Configure dead-letter destinations for undeliverable records

## Safety Ratings

```
safety_ratings:
  - "logs filter-log-events: GREEN — read-only log review"
  - "kinesis describe-stream-summary: GREEN — read-only stream details"
  - "s3api head-bucket: GREEN — read-only bucket check"
  - "update-application (sink config): YELLOW — modifies sink settings"
```

## Escalation Conditions

- Sink errors cause back-pressure that affects the entire pipeline
- Destination capacity cannot be increased to match output volume
- Exactly-once delivery is required but sink connector only supports at-least-once
- Sink errors cause data loss in downstream systems

## Data Sensitivity

- `aws logs filter-log-events`: HIGH — may contain output data in error messages
- Destination configuration: MEDIUM — reveals output targets
- Sink metrics: LOW — aggregated throughput data

## Prohibited Actions

- NEVER ignore sink errors — they cause data loss or back-pressure
- NEVER disable sink buffering without understanding throughput impact
- NEVER grant overly broad write permissions to destinations
- NEVER modify destination schema without coordinating with consumers

## Phase 3 — Rollback

1. If sink configuration was changed, revert to previous settings
2. If destination was modified, restore original configuration
3. Verify data integrity in the destination after sink fixes
4. Check for duplicate records if sink retried during errors
5. Document sink configuration and error handling strategy

## Common Issues

- symptoms: "ProvisionedThroughputExceededException writing to Kinesis output stream"
  diagnosis: "Output stream does not have enough shards for the write throughput."
  resolution: "Increase output stream shard count. Enable enhanced fan-out. Implement exponential backoff in sink."

## Output Format

```yaml
root_cause: "sink_connector — <specific_cause>"
evidence:
  - type: sink_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix sink connector"
  long_term: "Right-size destination capacity"
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
