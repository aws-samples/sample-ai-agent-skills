---
title: "C1 — Source Connector Errors"
description: "Diagnose Managed Flink source connector issues"
status: active
severity: HIGH
triggers: ["source error", "cannot read from Kinesis", "Kafka source error", "source deserialization"]
owner: devops-agent
objective: "Resolve source connector issues including Kinesis, Kafka, and S3 source errors"
context: "Source connectors read data from Kinesis streams, Kafka topics, S3, or custom sources. Errors include permissions, deserialization, and connectivity."
---

## Phase 1 — Triage

MUST:
- Check application logs for source errors: `aws logs filter-log-events --log-group-name /aws/kinesis-analytics/<app-name> --filter-pattern "source" --start-time <epoch-ms> --limit 50`
- Verify source stream/topic exists: `aws kinesis describe-stream-summary --stream-name <stream-name>`
- Check IAM role permissions for source access

SHOULD:
- Check source stream metrics (incoming records, iterator age)
- Verify deserialization schema matches source data format

MAY:
- Test source connectivity independently
- Check for source throttling: `aws cloudwatch get-metric-statistics --namespace AWS/Kinesis --metric-name ReadProvisionedThroughputExceeded --dimensions Name=StreamName,Value=<stream-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`

## Phase 2 — Remediate

MUST:
- Fix IAM role permissions for kinesis:GetRecords, kinesis:GetShardIterator, kinesis:DescribeStream
- Fix deserialization errors (match schema to actual data format)
- Ensure source stream/topic is active and has data

SHOULD:
- Configure error handling for malformed records
- Match source parallelism to number of shards/partitions

MAY:
- Implement custom deserialization for complex data formats
- Configure watermark strategy for event-time processing

## Safety Ratings

```
safety_ratings:
  - "logs filter-log-events: GREEN — read-only log review"
  - "kinesis describe-stream-summary: GREEN — read-only stream details"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "update-application (source config): YELLOW — modifies source settings"
```

## Escalation Conditions

- Source stream is in a different account and cross-account access is needed
- Deserialization errors affect a large percentage of records
- Source throttling cannot be resolved by increasing stream capacity
- Source connector requires VPC access that is not configured

## Data Sensitivity

- `aws logs filter-log-events`: HIGH — may contain source data samples in error messages
- `aws kinesis describe-stream-summary`: LOW — reveals stream metadata
- `aws cloudwatch get-metric-statistics`: LOW — aggregated metrics only

## Prohibited Actions

- NEVER grant kinesis:* permissions as a troubleshooting shortcut
- NEVER skip deserialization errors without logging them
- NEVER assume source data format is stable — validate schema
- NEVER modify source stream configuration without coordinating with producers

## Phase 3 — Rollback

1. If source configuration was changed, revert to previous settings
2. If IAM role was modified, restore original permissions
3. Verify application is reading from the correct source after changes
4. Check for data gaps caused by source connectivity issues
5. Document source configuration and error handling strategy

## Common Issues

- symptoms: "DeserializationException for records from Kinesis stream"
  diagnosis: "Source data format does not match the configured deserialization schema."
  resolution: "Verify data format (JSON, Avro, CSV). Update deserialization schema to match actual data. Add error handling for malformed records."

## Output Format

```yaml
root_cause: "source_connector — <specific_cause>"
evidence:
  - type: source_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix source connector"
  long_term: "Add schema validation"
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
