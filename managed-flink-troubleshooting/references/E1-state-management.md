---
title: "E1 — State Management"
description: "Diagnose Managed Flink state management and state backend issues"
status: active
severity: HIGH
triggers: ["state too large", "state backend error", "RocksDB error", "state corruption"]
owner: devops-agent
objective: "Resolve state management issues including state size, TTL, state backend configuration, and state migration"
context: "Flink state stores intermediate processing results. Managed Flink uses RocksDB state backend. State size affects checkpoint duration and memory usage."
---

## Phase 1 — Triage

MUST:
- Check state size via checkpoint metrics: `aws cloudwatch get-metric-statistics --namespace AWS/KinesisAnalytics --metric-name lastCheckpointSize --dimensions Name=Application,Value=<app-name> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check heap memory utilization: `aws cloudwatch get-metric-statistics --namespace AWS/KinesisAnalytics --metric-name heapMemoryUtilization --dimensions Name=Application,Value=<app-name> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Review logs for state-related errors: `aws logs filter-log-events --log-group-name /aws/kinesis-analytics/<app-name> --filter-pattern "state" --start-time <epoch-ms> --limit 50`

SHOULD:
- Check if state size is growing over time (unbounded state)
- Verify state TTL is configured for stateful operators

MAY:
- Review operator-level state sizes
- Check RocksDB metrics for compaction and write stalls

## Phase 2 — Remediate

MUST:
- Implement state TTL for time-bounded state (e.g., session windows)
- Reduce key cardinality if state has too many keys
- Increase KPUs if state exceeds available memory

SHOULD:
- Enable incremental checkpointing for large state
- Configure RocksDB options for the state access pattern

MAY:
- Implement custom state cleanup logic in process functions
- Use timer-based state expiration for complex cleanup patterns

## Safety Ratings

```
safety_ratings:
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "logs filter-log-events: GREEN — read-only log review"
  - "update-application (state config): YELLOW — modifies state backend settings"
  - "create-application-snapshot: YELLOW — captures current state"
```

## Escalation Conditions

- State size grows unbounded and causes out-of-memory errors
- State migration fails after code changes
- RocksDB compaction cannot keep up with write rate
- State corruption requires application restart from scratch

## Data Sensitivity

- `aws cloudwatch get-metric-statistics`: LOW — aggregated metrics only
- `aws logs filter-log-events`: HIGH — may contain state keys and values in errors
- Application snapshots: HIGH — contain full application state

## Prohibited Actions

- NEVER delete application snapshots without confirming they are no longer needed
- NEVER change state serialization format without a migration plan
- NEVER disable state TTL on production applications without understanding growth impact
- NEVER assume state size is stable — monitor growth trends

## Phase 3 — Rollback

1. Create snapshot before state changes: `aws kinesisanalyticsv2 create-application-snapshot --application-name <name> --snapshot-name pre-state-change`
2. If state configuration changes caused issues, restore from snapshot
3. If state TTL was added incorrectly, revert and restore from pre-change snapshot
4. Verify state integrity after rollback
5. Document state management strategy and TTL configuration

## Common Issues

- symptoms: "OutOfMemoryError: Java heap space after running for several hours"
  diagnosis: "State is growing unbounded. No TTL configured for stateful operators."
  resolution: "Configure state TTL for all stateful operators. Implement cleanup timers for session state."

## Output Format

```yaml
root_cause: "state_management — <specific_cause>"
evidence:
  - type: state_metric
    content: "<metric details>"
severity: HIGH
mitigation:
  immediate: "Fix state issue"
  long_term: "Implement state TTL"
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
