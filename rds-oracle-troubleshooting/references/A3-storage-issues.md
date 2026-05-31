---
title: "A3 — RDS Oracle Storage Issues"
description: "Diagnose storage capacity, performance, and configuration problems"
status: active
severity: HIGH
triggers:
  - "storage full"
  - "FreeStorageSpace"
  - "storage autoscaling"
  - "IOPS"
  - "storage type"
  - "gp2"
  - "gp3"
  - "io1"
  - "ORA-01654"
  - "ORA-01653"
owner: devops-agent
objective: "Resolve RDS Oracle storage capacity, performance, and type issues"
context: "RDS Oracle uses EBS only (no ASM, no raw devices). Storage types: gp2, gp3, io1, io2. Maximum 64 TiB. Storage autoscaling available. Storage can only be increased, never decreased. Storage modifications have a 6-hour cooldown."
---

## Phase 1 — Triage

MUST:
- Check current storage: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].{Storage:AllocatedStorage,Type:StorageType,IOPS:Iops,Throughput:StorageThroughput}'`
- Check free storage: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name FreeStorageSpace --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check storage autoscaling: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].MaxAllocatedStorage'`
- Check I/O metrics:
  - `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReadIOPS ...`
  - `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name WriteIOPS ...`
  - `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name ReadLatency ...`
  - `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name WriteLatency ...`

SHOULD:
- Check tablespace usage via SQL: `SELECT tablespace_name, ROUND(used_percent,2) AS pct_used FROM dba_tablespace_usage_metrics;`
- Check for large segments: `SELECT segment_name, segment_type, bytes/1024/1024 AS mb FROM dba_segments ORDER BY bytes DESC FETCH FIRST 20 ROWS ONLY;`
- Check pending storage modifications: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].PendingModifiedValues'`

MAY:
- Check for storage optimization opportunities (unused indexes, fragmented tables)
- Review Enhanced Monitoring for disk queue depth

## Phase 2 — Remediate

MUST:
- For storage full: increase allocated storage (cannot decrease)
  ```
  aws rds modify-db-instance \
    --db-instance-identifier <instance-id> \
    --allocated-storage <new-size-gb> \
    --apply-immediately
  ```
- For I/O bottleneck: consider storage type change (gp2→gp3 or io1/io2 for provisioned IOPS)
- Enable storage autoscaling if not set:
  ```
  aws rds modify-db-instance \
    --db-instance-identifier <instance-id> \
    --max-allocated-storage <max-gb>
  ```

SHOULD:
- Clean up Oracle tablespace: purge recyclebin, shrink segments
  ```sql
  PURGE DBA_RECYCLEBIN;
  -- Identify and shrink fragmented segments
  SELECT owner, segment_name, segment_type FROM dba_segments WHERE segment_type = 'TABLE' ORDER BY bytes DESC;
  ```
- Wait 6 hours between storage modifications (cooldown period)
- For gp3: configure IOPS and throughput independently of storage size

MAY:
- Migrate to io1/io2 for consistent high IOPS workloads
- Implement tablespace monitoring alerts

## Common Issues

- symptoms: "ORA-01654: unable to extend index / ORA-01653: unable to extend table"
  diagnosis: "Tablespace full. RDS storage may have space but Oracle tablespace needs extending."
  resolution: "Add datafile or resize existing datafile. If RDS storage is also full, increase allocated storage first."

- symptoms: "Storage modification stuck — 6-hour cooldown"
  diagnosis: "RDS enforces a 6-hour cooldown between storage modifications."
  resolution: "Wait for cooldown. Plan storage increases with sufficient headroom."

- symptoms: "High I/O latency on gp2"
  diagnosis: "gp2 IOPS scale with volume size (3 IOPS/GB, burst to 3000). Small volumes have low baseline IOPS."
  resolution: "Switch to gp3 (3000 baseline IOPS regardless of size) or io1/io2 for provisioned IOPS."

## Safety Ratings
- GREEN: describe-db-instances, CloudWatch FreeStorageSpace/ReadIOPS/WriteIOPS/ReadLatency/WriteLatency metrics, SELECT from dba_tablespace_usage_metrics/dba_segments — read-only inspection
- YELLOW: modify-db-instance (storage increase, type change), PURGE DBA_RECYCLEBIN — recoverable changes
- RED: delete-db-instance, dropping tablespaces — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Storage full causing ORA-01654/ORA-01653 errors"
- "Fix requires storage type change (may cause brief I/O pause)"
- "6-hour cooldown between storage modifications"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, tablespace details (reveal schema structure)
- HIGH: query results from dba_segments (reveal data distribution)
- MEDIUM: CloudWatch storage metrics, IOPS configuration

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix storage issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest shrinking datafiles without confirming space usage"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"

## Phase 3 — Rollback
- "Restore from snapshot if storage configuration change causes issues"
- "Storage increases cannot be reversed — plan increases with sufficient headroom"
- "Wait 6 hours between storage modifications (cooldown period)"
- "If storage type change causes I/O issues, revert to previous type after cooldown"

## Output Format

```yaml
root_cause: "storage — <specific_cause>"
evidence:
  - type: storage_config
    content: "<current storage type, size, IOPS>"
  - type: metrics
    content: "<FreeStorageSpace, IOPS, latency metrics>"
severity: HIGH
mitigation:
  immediate: "Increase storage or change storage type"
  long_term: "Enable storage autoscaling and implement tablespace monitoring"
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
  - "NEVER suggest making databases publicly accessible"
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest deleting automated backups"
