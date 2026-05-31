---
title: "E1 — Reader Replica Lag"
description: "Diagnose Aurora reader replica lag issues"
status: active
severity: HIGH
triggers:
  - "replica lag"
  - "AuroraReplicaLag"
  - "reader lag"
  - "stale reads"
  - "replication delay"
owner: devops-agent
objective: "Identify and resolve Aurora reader replica lag"
context: "Aurora readers share the same storage volume as the writer, so replication lag is typically sub-10ms. Lag occurs when the reader's page cache is stale and needs to apply redo log records. High write throughput on the writer, resource contention on the reader, or reader instance undersizing can increase lag."
---

## Phase 1 — Triage

MUST:
- Check replica lag metric:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name AuroraReplicaLag \
    --dimensions Name=DBInstanceIdentifier,Value=<reader-instance-id> \
    --start-time <start> --end-time <end> --period 60 --statistics Average Maximum
  ```
- Check writer write throughput:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name VolumeWriteIOPs \
    --dimensions Name=DBClusterIdentifier,Value=<cluster-id> ...
  ```
- Check reader CPU and memory:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name CPUUtilization \
    --dimensions Name=DBInstanceIdentifier,Value=<reader-instance-id> ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeableMemory \
    --dimensions Name=DBInstanceIdentifier,Value=<reader-instance-id> ...
  ```
- For Aurora MySQL — check replication status:
  ```sql
  SHOW SLAVE STATUS\G
  -- Look for Seconds_Behind_Master
  ```
- For Aurora PostgreSQL — check replication lag:
  ```sql
  SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
  ```

SHOULD:
- Compare reader instance class with writer instance class
- Check if reader is running heavy queries that compete with replication apply
- Check for long-running transactions on the reader that block redo apply

MAY:
- Check Performance Insights on the reader for wait events
- Review if reader is used for both read queries and replication (resource contention)

## Phase 2 — Remediate

MUST:
- If reader is undersized: scale up reader instance class to match or approach writer class
- If reader is overloaded with queries: add more readers to distribute load
- If writer has extreme write throughput: optimize write patterns

SHOULD:
- Use separate readers for different workloads (custom endpoints)
- Monitor AuroraReplicaLag with CloudWatch alarms (threshold: depends on application tolerance)
- Ensure reader instance class has sufficient memory for page cache

MAY:
- Consider Aurora Global Database if cross-region read lag is the concern
- Implement application-level staleness tolerance for read queries

## Common Issues

- symptoms: "AuroraReplicaLag consistently > 100ms"
  diagnosis: "Reader instance undersized or overloaded with queries."
  resolution: "Scale up reader. Add more readers. Distribute query load."

- symptoms: "Replica lag spikes during batch writes"
  diagnosis: "High write throughput on writer causes reader to fall behind applying redo."
  resolution: "Throttle batch writes. Scale up reader. Increase reader memory."

- symptoms: "Replica lag on one reader but not others"
  diagnosis: "Specific reader is undersized or running heavy queries."
  resolution: "Scale up the affected reader. Move heavy queries to a dedicated reader."

## Safety Ratings
- GREEN: describe-db-clusters, describe-db-instances, CloudWatch AuroraReplicaLag/VolumeWriteIOPs/CPUUtilization/FreeableMemory metrics, SHOW SLAVE STATUS, pg_last_xact_replay_timestamp() — read-only inspection
- YELLOW: modify-db-instance (scale up reader), modify-db-cluster-endpoint — recoverable changes
- RED: delete-db-instance, force-failover — destructive or high-impact operations in production

## Escalation Conditions
- "Database serves production traffic"
- "Replica lag causing stale reads affecting application correctness"
- "Fix requires scaling up reader instances"
- "Fix requires failover of Aurora cluster"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, replication status details
- HIGH: query results from SHOW SLAVE STATUS (contain replication configuration)
- MEDIUM: CloudWatch replica lag metrics, CPU/memory metrics, write throughput

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix lag issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest force-failover in production without confirming application readiness"
- "NEVER suggest stopping replication to fix lag — this will make it worse"

## Phase 3 — Rollback
- "Restore from snapshot if reader configuration change causes issues"
- "If scaling up a reader causes issues, scale back down after confirming"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "reader_lag — <specific_cause>"
evidence:
  - type: cloudwatch
    content: "<AuroraReplicaLag, CPU, memory metrics>"
  - type: replication_status
    content: "<replication status from SQL>"
  - type: write_throughput
    content: "<writer VolumeWriteIOPs>"
severity: HIGH
mitigation:
  immediate: "Scale up reader or reduce reader query load"
  long_term: "Implement reader monitoring and workload isolation"
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
