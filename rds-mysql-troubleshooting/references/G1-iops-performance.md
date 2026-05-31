---
title: "G1 — Storage IOPS Performance"
description: "Diagnose storage IOPS throttling and I/O latency on RDS MySQL"
status: active
severity: HIGH
triggers:
  - "IOPS"
  - "I/O latency"
  - "storage slow"
  - "disk throughput"
  - "gp2 burst"
---

## Phase 1 — Triage

MUST:
- Check IOPS metrics: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReadIOPS --dimensions Name=DBInstanceIdentifier,Value=<id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check latency: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReadLatency ...` and `WriteLatency`
- Check storage type and provisioned IOPS: `aws rds describe-db-instances --db-instance-identifier <id> --query 'DBInstances[0].[StorageType,Iops,AllocatedStorage]'`
- Check queue depth: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DiskQueueDepth ...`

SHOULD:
- For gp2: check burst credit balance — `BurstBalance` metric (< 20% = throttling risk)
- For gp3: baseline is 3000 IOPS / 125 MiB/s — check if workload exceeds baseline
- For io1/io2: verify provisioned IOPS matches workload needs
- Check if InnoDB is doing excessive disk reads (low buffer pool hit ratio)

## Phase 2 — Remediate

- gp2 throttling: migrate to gp3 (better baseline) or io1/io2 for consistent IOPS
- gp3: increase provisioned IOPS beyond 3000 baseline if needed
- io1/io2: increase provisioned IOPS (max 256,000 for io2)
- Reduce I/O: optimize queries, increase buffer pool, add indexes
- Storage modification: `aws rds modify-db-instance --db-instance-identifier <id> --storage-type gp3 --iops 6000 --apply-immediately`

## Safety Ratings
- GREEN: CloudWatch ReadIOPS/WriteIOPS/ReadLatency/WriteLatency/DiskQueueDepth metrics, describe-db-instances — read-only inspection
- YELLOW: modify-db-instance (storage type/IOPS change) — recoverable but 6-hour cooldown between changes
- RED: delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "I/O latency causing application timeouts"
- "Fix requires storage type change (6-hour cooldown)"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings
- MEDIUM: CloudWatch I/O metrics, storage configuration

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix I/O issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"

## Phase 3 — Rollback
- "Restore from snapshot if storage configuration change causes issues"
- "Wait 6 hours between storage modifications (cooldown period)"
- "Storage increases cannot be reversed — plan increases with sufficient headroom"

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
