---
title: "Z1 — General Aurora Troubleshooting (Catch-All)"
description: "Fallback SOP for Aurora issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown Aurora issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Check cluster status: `aws rds describe-db-clusters --db-cluster-identifier <cluster-id>`
- Check all instance status:
  ```
  aws rds describe-db-instances --filters Name=db-cluster-id,Values=<cluster-id> \
    --query 'DBInstances[].{Id:DBInstanceIdentifier,Status:DBInstanceStatus,Class:DBInstanceClass,AZ:AvailabilityZone,Role:DBInstanceIdentifier}'
  ```
- Check recent events:
  ```
  aws rds describe-events --source-identifier <cluster-id> --source-type db-cluster --duration 1440
  aws rds describe-events --source-identifier <writer-instance-id> --source-type db-instance --duration 1440
  ```
- Check key CloudWatch metrics:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name CPUUtilization \
    --dimensions Name=DBInstanceIdentifier,Value=<writer-instance-id> \
    --start-time <start> --end-time <end> --period 300 --statistics Average
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeableMemory ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name VolumeBytesUsed \
    --dimensions Name=DBClusterIdentifier,Value=<cluster-id> ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections ...
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name AuroraReplicaLag ...
  ```
- Check engine-level health:
  - Aurora MySQL:
    ```sql
    SHOW STATUS LIKE 'Threads_connected';
    SHOW STATUS LIKE 'Threads_running';
    SHOW ENGINE INNODB STATUS\G
    ```
  - Aurora PostgreSQL:
    ```sql
    SELECT count(*) AS total_connections FROM pg_stat_activity;
    SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
    SELECT wait_event_type, wait_event, count(*) FROM pg_stat_activity
    WHERE state = 'active' AND wait_event IS NOT NULL GROUP BY wait_event_type, wait_event ORDER BY count DESC;
    ```

SHOULD:
- Check parameter group status:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].DBClusterParameterGroup'
  ```
- Check pending modifications:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].PendingModifiedValues'
  ```
- Check pending maintenance:
  ```
  aws rds describe-pending-maintenance-actions --resource-identifier <cluster-arn>
  ```
- Check Performance Insights for DB load:
  ```
  aws pi get-resource-metrics --service-type RDS --identifier db-<resource-id> \
    --metric-queries '[{"Metric":"db.load.avg"}]' --start-time <start> --end-time <end> --period-in-seconds 300
  ```

MAY:
- Check Enhanced Monitoring for OS-level metrics
- Check CloudTrail for recent configuration changes
- Check if cluster is part of a Global Database: `aws rds describe-global-clusters`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Cluster creation/config/storage → Cluster (A1-A4)
- CPU, memory, I/O, query performance → Performance (B1-B4)
- Connection failures, endpoints, proxy, limits → Connectivity (C1-C4)
- Writer/reader failover, DNS → Failover (D1-D3)
- Reader lag, binlog, logical replication → Replication (E1-E3)
- Serverless scaling, cold start, capacity → Serverless (F1-F3)
- Global Database lag, failover → Global Database (G1-G3)
- Backup, backtrack, clone → Backup & Recovery (H1-H3)
- Encryption, IAM auth → Security (I1-I2)
- DMS, MySQL/PostgreSQL migration → Migration (J1-J2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all cluster and instance configurations systematically
- Review CloudTrail for recent API calls affecting the cluster
- Check error logs:
  - Aurora MySQL: `aws rds download-db-log-file-portion --db-instance-identifier <instance-id> --log-file-name error/mysql-error-running.log --number-of-lines 500`
  - Aurora PostgreSQL: `aws rds download-db-log-file-portion --db-instance-identifier <instance-id> --log-file-name error/postgresql.log --number-of-lines 500`
- Verify network connectivity and security group rules

SHOULD:
- Check AWS Health Dashboard for Aurora service events
- Compare with a known-good cluster configuration
- Check for recent changes (parameter group, security group, instance class)

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Safety Ratings
- GREEN: describe-db-clusters, describe-db-instances, describe-events, CloudWatch metrics, SHOW STATUS/ENGINE INNODB STATUS, pg_stat_activity, describe-pending-maintenance-actions — read-only inspection
- YELLOW: modify-db-instance, modify-db-cluster — recoverable configuration changes
- RED: delete-db-cluster, delete-db-instance, force-failover — destructive or high-impact operations in production

## Escalation Conditions
- "Database serves production traffic"
- "Root cause cannot be identified — escalate to AWS Support"
- "Fix requires parameter group change that needs reboot"
- "Fix requires failover of Aurora cluster"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, error log contents (may contain SQL text and sensitive data)
- HIGH: SHOW ENGINE INNODB STATUS, pg_stat_activity (contain active SQL text and connection details)
- MEDIUM: CloudWatch metrics, cluster configuration, pending maintenance actions

## Prohibited Actions
- "NEVER suggest deleting a database cluster to fix issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest force-failover in production without confirming application readiness"

## Phase 3 — Rollback
- "Restore from snapshot if configuration change causes issues"
- "Revert parameter group changes and reboot if needed"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<cluster|performance|connectivity|failover|replication|serverless|global|backup|security|migration|unknown>"
investigation_path: "cluster status → events → metrics → engine health → <domain_classification>"
evidence:
  - type: cluster_config
    content: "<cluster configuration summary>"
  - type: cloudwatch
    content: "<key metrics>"
  - type: engine_health
    content: "<engine-level diagnostics>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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
