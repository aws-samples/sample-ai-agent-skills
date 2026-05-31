---
title: "Z1 — General RDS PostgreSQL Troubleshooting"
description: "Catch-all runbook for RDS PostgreSQL issues"
status: active
severity: MEDIUM
triggers:
  - "RDS PostgreSQL"
  - "something is wrong"
  - "not working"
---

## Phase 1 — General Triage

MUST:
- Get instance overview: `aws rds describe-db-instances --db-instance-identifier <id>`
- Check recent events: `aws rds describe-events --source-identifier <id> --source-type db-instance --duration 1440`
- Check key metrics: CPUUtilization, FreeableMemory, FreeStorageSpace, DatabaseConnections, TransactionLogsDiskUsage, ReplicaLag
- Check error log: `aws rds download-db-log-file-portion --db-instance-identifier <id> --log-file-name error/postgresql.log`

SHOULD:
- Check Performance Insights for DB load
- Check pg_stat_activity for active queries
- Review CloudTrail for recent API changes

## Phase 2 — Route to Specific Runbook

Based on findings, route to:
- Launch → A1, Vacuum/bloat → B1, Connections → B2, pg_stat → B3
- Connectivity → C1, Parameters → D1
- Replication → E1, Logical replication → E2
- Backup/PITR → F1, Extensions → G1, Upgrades → H1
- Proxy → I1, Encryption → J1

## Safety Ratings
- GREEN: describe-db-instances, describe-events, CloudWatch metrics, SELECT from pg_stat_activity, SHOW settings — read-only inspection
- YELLOW: modify-db-instance, modify-db-parameter-group — recoverable configuration changes
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Root cause cannot be identified — escalate to AWS Support"
- "Fix requires parameter group change that needs reboot"
- "Fix requires failover of Multi-AZ instance"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, error log contents (may contain SQL text)
- HIGH: pg_stat_activity output (contains active SQL text and session details)
- MEDIUM: CloudWatch metrics, instance configuration, pending maintenance actions

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"
- "NEVER suggest force-failover in production without confirming application readiness"

## Phase 3 — Rollback
- "Restore from snapshot if configuration change causes issues"
- "Revert parameter group changes and reboot if needed"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

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
  - "NEVER suggest making databases publicly accessible"
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest deleting automated backups"
