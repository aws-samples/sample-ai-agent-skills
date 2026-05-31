---
title: "J1 — DMS to Aurora Migration"
description: "Diagnose AWS DMS migration issues when migrating to Aurora"
status: active
severity: HIGH
triggers:
  - "DMS"
  - "Database Migration Service"
  - "migration task"
  - "replication instance"
  - "CDC"
  - "full load"
owner: devops-agent
objective: "Identify and resolve DMS migration issues when migrating to Aurora"
context: "AWS DMS supports migration to Aurora MySQL and Aurora PostgreSQL from various sources. Common issues include replication instance sizing, endpoint configuration, CDC (change data capture) setup, LOB handling, and data type mapping. DMS uses source and target endpoints with a replication instance."
---

## Phase 1 — Triage

MUST:
- Check DMS replication task status:
  ```
  aws dms describe-replication-tasks --filters Name=replication-task-id,Values=<task-id> \
    --query 'ReplicationTasks[0].{Status:Status,StopReason:StopReason,LastError:LastFailureMessage}'
  ```
- Check replication instance status:
  ```
  aws dms describe-replication-instances --filters Name=replication-instance-id,Values=<instance-id> \
    --query 'ReplicationInstances[0].{Status:ReplicationInstanceStatus,Class:ReplicationInstanceClass}'
  ```
- Check target endpoint (Aurora) connectivity:
  ```
  aws dms test-connection --replication-instance-arn <instance-arn> --endpoint-arn <target-endpoint-arn>
  aws dms describe-connections --filter Name=endpoint-arn,Values=<target-endpoint-arn>
  ```
- Check task table statistics:
  ```
  aws dms describe-table-statistics --replication-task-arn <task-arn>
  ```

SHOULD:
- Check DMS task logs in CloudWatch:
  ```
  aws logs filter-log-events --log-group-name dms-tasks-<task-id>
  ```
- Verify target endpoint configuration:
  ```
  aws dms describe-endpoints --filter Name=endpoint-arn,Values=<target-endpoint-arn>
  ```
- Check for data validation errors:
  ```
  aws dms describe-table-statistics --replication-task-arn <task-arn> \
    --query 'TableStatistics[?ValidationState!=`Validated`]'
  ```

MAY:
- Check replication instance CloudWatch metrics (CPU, memory, storage)
- Review table mapping rules for correctness

## Phase 2 — Remediate

MUST:
- For connection failures: verify security groups allow DMS replication instance to connect to Aurora on the database port
- For task errors: check LastFailureMessage and CloudWatch logs for specific error
- For CDC issues: ensure source database has appropriate logging enabled (binlog for MySQL, logical replication for PostgreSQL)

SHOULD:
- Size replication instance appropriately (at least as large as the source database workload)
- Use full LOB mode only when necessary (limited LOB mode is faster)
- Enable data validation for critical tables
- Use the Aurora cluster endpoint (writer) as the target endpoint

MAY:
- Use DMS pre-migration assessment to identify potential issues
- Implement table-level parallelism for faster full load

## Common Issues

- symptoms: "DMS task failed with connection error to Aurora target"
  diagnosis: "Security group, VPC routing, or endpoint configuration issue."
  resolution: "Verify security groups. Use cluster writer endpoint. Check VPC connectivity."

- symptoms: "CDC replication lag increasing"
  diagnosis: "Replication instance undersized or Aurora target cannot keep up with writes."
  resolution: "Scale up replication instance. Scale up Aurora writer. Optimize table mappings."

- symptoms: "Data type conversion errors"
  diagnosis: "Source data types not compatible with Aurora target."
  resolution: "Use DMS transformation rules. Check data type mapping documentation."

## Safety Ratings
- GREEN: describe-replication-tasks, describe-replication-instances, describe-endpoints, test-connection, describe-table-statistics, CloudWatch Logs — read-only inspection
- YELLOW: modify-replication-task, modify-replication-instance — recoverable configuration changes
- RED: delete-replication-task, delete-replication-instance, delete-db-cluster — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "DMS migration task failed and data consistency is at risk"
- "Fix requires modifying Aurora target cluster configuration"
- "CDC replication lag increasing and cutover window is approaching"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials in DMS endpoints, connection strings, replication instance configuration
- HIGH: DMS task logs (may contain SQL statements and data samples)
- MEDIUM: table migration statistics, replication instance metrics

## Prohibited Actions
- "NEVER suggest deleting a database cluster to fix DMS issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest cutting over to Aurora target without validating data consistency"

## Phase 3 — Rollback
- "Restore from snapshot if DMS migration causes data issues on Aurora target"
- "If DMS task fails, do NOT delete the task — stop and investigate first"
- "Revert Aurora target parameter changes if they cause DMS compatibility issues"
- "If migration cutover fails, revert application to source database"

## Output Format

```yaml
root_cause: "dms_migration — <specific_cause>"
evidence:
  - type: task_status
    content: "<DMS task status and error>"
  - type: connection_test
    content: "<endpoint connectivity test result>"
  - type: table_stats
    content: "<table migration statistics>"
severity: HIGH
mitigation:
  immediate: "Fix DMS task error and resume migration"
  long_term: "Implement migration monitoring and validation"
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
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling encryption"
  - "NEVER force failover without understanding impact"
