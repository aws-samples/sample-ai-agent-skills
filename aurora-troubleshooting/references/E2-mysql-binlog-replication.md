---
title: "E2 — Aurora MySQL Binary Log Replication"
description: "Diagnose Aurora MySQL binlog replication issues"
status: active
severity: HIGH
triggers:
  - "binlog replication"
  - "binary log"
  - "binlog_format"
  - "AuroraBinlogReplicaLag"
  - "external replication"
  - "binlog position"
owner: devops-agent
objective: "Identify and resolve Aurora MySQL binary log replication issues"
context: "Aurora MySQL supports binlog replication for replicating to external MySQL targets, other Aurora clusters, or for CDC. Binlog must be explicitly enabled via cluster parameter group (binlog_format). Enabling binlog adds write latency. Binlog replication is separate from Aurora's native storage-level replication."
---

## Phase 1 — Triage

MUST:
- Check if binlog is enabled:
  ```sql
  SHOW VARIABLES LIKE 'binlog_format';
  SHOW VARIABLES LIKE 'log_bin';
  ```
- Check binlog replication lag:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name AuroraBinlogReplicaLag \
    --dimensions Name=DBClusterIdentifier,Value=<cluster-id> \
    --start-time <start> --end-time <end> --period 300 --statistics Average
  ```
- Check replication status on the replica:
  ```sql
  SHOW SLAVE STATUS\G
  -- Key fields: Slave_IO_Running, Slave_SQL_Running, Seconds_Behind_Master, Last_Error
  ```
- Check binlog retention:
  ```sql
  CALL mysql.rds_show_configuration;
  -- Look for binlog retention hours
  ```

SHOULD:
- Check cluster parameter group for binlog settings:
  ```
  aws rds describe-db-cluster-parameters --db-cluster-parameter-group-name <cluster-param-group> \
    --query 'Parameters[?ParameterName==`binlog_format`]'
  ```
- Check binary log files:
  ```sql
  SHOW BINARY LOGS;
  SHOW MASTER STATUS;
  ```
- Verify replication user permissions:
  ```sql
  SHOW GRANTS FOR 'repl_user'@'%';
  -- Needs REPLICATION SLAVE, REPLICATION CLIENT
  ```

MAY:
- Check for binlog-related write latency impact
- Review GTID configuration if using GTID-based replication

## Phase 2 — Remediate

MUST:
- For replication stopped: check Last_Error in SHOW SLAVE STATUS and resolve
- For binlog not enabled: set `binlog_format` in cluster parameter group (requires reboot)
- For binlog retention: set retention hours: `CALL mysql.rds_set_configuration('binlog retention hours', 24);`

SHOULD:
- Use `binlog_format = ROW` for most reliable replication
- Set appropriate binlog retention (default is NULL = no retention beyond crash recovery)
- Monitor AuroraBinlogReplicaLag with CloudWatch alarms

MAY:
- Consider GTID-based replication for easier failover management
- Use DMS instead of native binlog replication for complex migration scenarios

## Common Issues

- symptoms: "Binlog replication lag increasing"
  diagnosis: "Replica cannot keep up with writer binlog production."
  resolution: "Scale up replica. Optimize write patterns. Check network between source and replica."

- symptoms: "Slave_SQL_Running = No"
  diagnosis: "SQL thread stopped due to replication error (duplicate key, missing row)."
  resolution: "Check Last_Error. Skip error if safe: CALL mysql.rds_skip_repl_error; or fix data inconsistency."

- symptoms: "Binary logs purged before replica consumed them"
  diagnosis: "Binlog retention too short for replica lag."
  resolution: "Increase binlog retention hours. Reduce replica lag."

## Safety Ratings
- GREEN: SHOW VARIABLES, SHOW SLAVE STATUS, SHOW BINARY LOGS, SHOW MASTER STATUS, describe-db-cluster-parameters, CloudWatch AuroraBinlogReplicaLag — read-only inspection
- YELLOW: modify-db-cluster-parameter-group (binlog_format), CALL mysql.rds_set_configuration, CALL mysql.rds_skip_repl_error — recoverable but may require reboot or affect replication
- RED: delete-db-cluster, force-failover, dropping replication without backup — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires enabling binlog_format (needs cluster parameter group change and reboot)"
- "Fix requires skipping replication errors (potential data inconsistency)"
- "Binlog replication lag increasing and affecting downstream consumers"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, replication user credentials
- HIGH: SHOW SLAVE STATUS output (contains replication configuration and connection details)
- MEDIUM: binlog configuration, retention settings, CloudWatch replication lag metrics

## Prohibited Actions
- "NEVER suggest deleting a database cluster to fix replication issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest force-failover in production without confirming application readiness"
- "NEVER suggest skipping replication errors without understanding the data impact"

## Phase 3 — Rollback
- "Restore from snapshot if binlog parameter change causes issues"
- "Revert parameter group changes (binlog_format) and reboot if needed"
- "If replication skip causes data inconsistency, rebuild replica from snapshot"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "binlog_replication — <specific_cause>"
evidence:
  - type: replication_status
    content: "<SHOW SLAVE STATUS output>"
  - type: cloudwatch
    content: "<AuroraBinlogReplicaLag metric>"
  - type: binlog_config
    content: "<binlog_format, retention settings>"
severity: HIGH
mitigation:
  immediate: "Fix replication error or adjust binlog retention"
  long_term: "Monitor binlog lag and implement replication alerting"
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
