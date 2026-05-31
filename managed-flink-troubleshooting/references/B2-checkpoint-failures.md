---
title: "B2 — Checkpoint Failures"
description: "Diagnose Flink checkpoint failures in Managed Flink applications"
status: active
severity: CRITICAL
triggers: ["checkpoint failed", "checkpoint timeout", "checkpoint too slow", "state too large"]
owner: devops-agent
objective: "Resolve checkpoint failures including timeout, state size, and back-pressure issues"
context: "Checkpoints are critical for fault tolerance. Failed checkpoints mean the application cannot recover from failures without data loss."
---

## Phase 1 — Triage

MUST:
- Check checkpoint metrics: `aws cloudwatch get-metric-statistics --namespace AWS/KinesisAnalytics --metric-name numberOfFailedCheckpoints --dimensions Name=Application,Value=<app-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check checkpoint duration: `aws cloudwatch get-metric-statistics --namespace AWS/KinesisAnalytics --metric-name lastCheckpointDuration --dimensions Name=Application,Value=<app-name> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check checkpoint size: `aws cloudwatch get-metric-statistics --namespace AWS/KinesisAnalytics --metric-name lastCheckpointSize --dimensions Name=Application,Value=<app-name> --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Review application logs for checkpoint errors: `aws logs filter-log-events --log-group-name /aws/kinesis-analytics/<app-name> --filter-pattern "checkpoint" --start-time <epoch-ms> --limit 50`
- Check heap memory utilization for state pressure

MAY:
- Check back-pressure metrics that may delay checkpoint barriers
- Review operator-level checkpoint alignment times

## Phase 2 — Remediate

MUST:
- Increase checkpoint timeout if checkpoints are timing out
- Reduce state size if checkpoints are too large (use TTL, reduce key cardinality)
- Fix back-pressure issues that delay checkpoint barrier alignment

SHOULD:
- Enable incremental checkpointing for RocksDB state backend
- Configure checkpoint interval appropriate for state size

MAY:
- Tune RocksDB configuration for better checkpoint performance
- Implement state TTL to automatically expire old state entries

## Safety Ratings

```
safety_ratings:
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "logs filter-log-events: GREEN — read-only log review"
  - "update-application (checkpoint config): YELLOW — modifies checkpoint settings"
  - "create-application-snapshot: YELLOW — triggers state snapshot"
```

## Escalation Conditions

- Checkpoints fail consistently and application cannot recover from failures
- State size grows unbounded and exceeds available memory
- Checkpoint duration exceeds the checkpoint interval
- Back-pressure causes cascading checkpoint failures across operators

## Data Sensitivity

- `aws cloudwatch get-metric-statistics`: LOW — aggregated metrics only
- `aws logs filter-log-events`: HIGH — may contain state details and error context
- Checkpoint state: HIGH — contains application processing state

## Prohibited Actions

- NEVER disable checkpointing to avoid checkpoint failures
- NEVER increase checkpoint interval beyond acceptable data loss window
- NEVER ignore growing checkpoint sizes — they indicate state management issues
- NEVER assume checkpoint failures are transient without investigating

## Phase 3 — Rollback

1. If checkpoint configuration was changed, revert to previous settings
2. If state TTL was added, verify it doesn't expire needed state
3. Create a snapshot before making further changes: `aws kinesisanalyticsv2 create-application-snapshot --application-name <name> --snapshot-name pre-fix-<timestamp>`
4. Verify checkpoints succeed after configuration changes
5. Document checkpoint configuration and state management strategy

## Common Issues

- symptoms: "numberOfFailedCheckpoints increasing, lastCheckpointDuration exceeds timeout"
  diagnosis: "State size is too large for the configured checkpoint timeout."
  resolution: "Increase checkpoint timeout. Enable incremental checkpointing. Implement state TTL to reduce state size."

## Output Format

```yaml
root_cause: "checkpoint_failure — <specific_cause>"
evidence:
  - type: checkpoint_metric
    content: "<metric details>"
severity: CRITICAL
mitigation:
  immediate: "Fix checkpoint failures"
  long_term: "Implement state management"
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
