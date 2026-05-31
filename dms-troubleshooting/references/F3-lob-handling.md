---
title: "F3 — LOB Handling"
description: "Diagnose LOB (Large Object) handling issues in DMS migrations"
status: active
severity: MEDIUM
triggers:
  - "LOB error"
  - "BLOB truncated"
  - "CLOB missing"
  - "large object"
  - "LOB slow"
owner: devops-agent
objective: "Configure LOB handling correctly for data integrity and acceptable performance"
context: "LOB columns (BLOB, CLOB, TEXT, BYTEA, etc.) require special handling in DMS. Full LOB mode is accurate but slow. Limited LOB mode is fast but truncates. Inline LOB mode balances both. Wrong LOB settings cause data loss or extreme slowness."
---

## Phase 1 — Triage

MUST:
- Check task LOB settings: `aws dms describe-replication-tasks --filters Name=replication-task-id,Values=<task-id> --query 'ReplicationTasks[*].ReplicationTaskSettings'`
- Check for LOB-related errors: `aws dms describe-replication-tasks --filters Name=replication-task-id,Values=<task-id> --query 'ReplicationTasks[*].LastFailureMessage'`
- Check table statistics for tables with LOB columns: `aws dms describe-table-statistics --replication-task-arn <task-arn>`
- Check throughput during LOB table processing: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name FullLoadThroughputRowsSource --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Identify which tables have LOB columns and their max sizes
- Check if LobMaxSize is set appropriately for limited LOB mode
- Review if InlineLobMaxSize is configured for inline LOB mode

MAY:
- Query source database for actual max LOB sizes per table
- Compare migration speed with and without LOB tables

## Phase 2 — Remediate

MUST:
- Choose appropriate LOB mode based on data requirements:
  - Full LOB: when all LOB data must be preserved exactly (slowest)
  - Limited LOB: when LOBs can be truncated at MaxLobSize (fastest)
  - Inline LOB: when most LOBs are small with few large ones (balanced)
- Set LobMaxSize to accommodate the largest LOB values (for limited mode)

SHOULD:
- Use inline LOB mode with InlineLobMaxSize for best performance
- Separate LOB-heavy tables into their own task with appropriate settings
- Increase replication instance storage for full LOB mode

MAY:
- Migrate LOB data in a separate pass after initial full load
- Use table-specific LOB settings via task table mappings

## Common Issues

- symptoms: "LOB data truncated on target"
  diagnosis: "Limited LOB mode with LobMaxSize smaller than actual LOB values."
  resolution: "Increase LobMaxSize or switch to full/inline LOB mode."

- symptoms: "Migration extremely slow on LOB tables"
  diagnosis: "Full LOB mode reading each LOB individually from source."
  resolution: "Switch to inline LOB mode. Set InlineLobMaxSize to cover most LOBs."

- symptoms: "Out of memory during LOB processing"
  diagnosis: "Very large LOBs exceeding replication instance memory."
  resolution: "Scale up instance. Use limited LOB mode with appropriate max size."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Switch LOB mode (full/limited/inline) | YELLOW | May truncate data — verify requirements |
| Set LobMaxSize for limited mode | YELLOW | Truncates LOBs exceeding max — verify sizes |
| Set InlineLobMaxSize for inline mode | GREEN | Optimization — non-destructive for small LOBs |
| Separate LOB-heavy tables into own task | GREEN | Task restructuring — non-destructive |
| Scale up replication instance storage | GREEN | Storage increase — no downtime |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- LOB truncation causes data loss in critical columns
- LOB processing slowness delays migration timeline

## Data Sensitivity

- **Classification: HIGH**
- LOB columns may contain documents, images, or other sensitive binary data
- LOB size analysis reveals data characteristics of source database
- Task settings expose data handling configuration
- Table-level LOB details reveal which tables contain large objects

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest using limited LOB mode without confirming acceptable truncation for all LOB columns
- **NEVER** suggest ignoring LOB truncation warnings in production migrations

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Changed LOB mode | Revert LOB mode and reload affected tables |
| Set LobMaxSize | Increase LobMaxSize and reload if data was truncated |
| Separated LOB tables into own task | Consolidate tasks if separation causes management overhead |

## Output Format

```yaml
root_cause: "lob_handling — <specific_cause>"
evidence:
  - type: lob_settings
    content: "<current LOB mode and size settings>"
  - type: affected_tables
    content: "<tables with LOB columns>"
severity: MEDIUM
mitigation:
  immediate: "Adjust LOB mode and size settings"
  long_term: "Profile LOB sizes before migration and choose optimal settings"
```

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
  - "NEVER suggest disabling SSL for replication endpoints"
  - "NEVER suggest public replication instances"
  - "NEVER suggest deleting replication tasks without data verification"
