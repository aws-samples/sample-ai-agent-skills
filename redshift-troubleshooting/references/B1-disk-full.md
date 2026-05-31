---
title: "B1 — Disk Full"
description: "Diagnose and resolve disk space issues in Redshift"
status: active
severity: CRITICAL
triggers:
  - "disk full"
  - "disk space"
  - "PercentageDiskSpaceUsed"
  - "no space left"
  - "storage full"
  - "cannot insert"
owner: devops-agent
objective: "Identify disk space consumption and free space or expand storage"
context: "DC2 nodes have fixed local SSD storage. RA3 nodes use managed storage (virtually unlimited). Disk full on DC2 blocks all write operations. High disk usage on RA3 indicates local cache pressure, not a hard limit. Deleted rows consume space until VACUUM reclaims it."
---

## Phase 1 — Triage

MUST:
- Check disk usage: `aws cloudwatch get-metric-statistics --namespace AWS/Redshift --metric-name PercentageDiskSpaceUsed --dimensions Name=ClusterIdentifier,Value=<cluster-id> --start-time $(date -u -d '6 hours ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 300 --statistics Maximum`
- Check node type (DC2 vs RA3): `aws redshift describe-clusters --cluster-identifier <cluster-id> --query 'Clusters[0].NodeType'`
- Check table sizes: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT schema, \"table\", size AS size_mb, pct_used, empty, unsorted FROM svv_table_info ORDER BY size DESC LIMIT 20"`
- Check for reclaimable space (deleted rows): `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT schema, \"table\", size, empty FROM svv_table_info WHERE empty > 20 ORDER BY size DESC LIMIT 20"`

SHOULD:
- Check per-node disk usage for skew: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT owner, host, diskno, used, capacity FROM stv_partitions WHERE used > capacity * 0.8 ORDER BY used DESC"`
- Check for orphaned temporary tables
- Check for large staging tables that should have been dropped

MAY:
- Check snapshot storage consumption
- Review data retention policies for old data
- Check for tables with excessive column count (wide tables use more space)

## Phase 2 — Remediate

MUST:
- If DC2 with high empty percentage: run `VACUUM DELETE` on tables with empty > 20%
- If DC2 truly full: add nodes (elastic resize) or migrate to RA3
- If RA3: disk usage reflects cache, not a hard limit — investigate if queries are failing
- Drop unnecessary tables, staging tables, or temporary tables

SHOULD:
- Run VACUUM DELETE on all tables with significant deleted rows
- Implement data lifecycle policies (drop old partitions, archive to S3)
- Set up CloudWatch alarm on PercentageDiskSpaceUsed at 75% and 90%

MAY:
- Consider column encoding changes to reduce storage
- Use UNLOAD to archive old data to S3 before deleting
- Evaluate migration from DC2 to RA3 for managed storage

## Common Issues

- symptoms: "PercentageDiskSpaceUsed at 100%, all writes failing"
  diagnosis: "DC2 cluster local storage is full. Deleted rows not reclaimed."
  resolution: "Run VACUUM DELETE immediately. Drop unused tables. Add nodes via elastic resize if needed."

- symptoms: "PercentageDiskSpaceUsed at 90% on RA3 nodes"
  diagnosis: "RA3 local cache is heavily used but storage is not actually limited."
  resolution: "RA3 uses managed storage — this is cache pressure, not a hard limit. Queries should still work. Monitor for actual failures."

## Output Format

```yaml
root_cause: "Disk full — <specific_cause>"
evidence:
  - type: cloudwatch_metric
    content: "<PercentageDiskSpaceUsed data>"
  - type: system_table
    content: "<svv_table_info showing large tables and empty percentages>"
severity: CRITICAL
mitigation:
  immediate: "VACUUM DELETE, drop unused tables, or elastic resize"
  long_term: "Migrate to RA3, implement data lifecycle policies, set up disk alerts"
```


## Safety Ratings
```
safety_ratings:
  - "Check CloudWatch disk metrics: GREEN — read-only monitoring query"
  - "Query svv_table_info: GREEN — read-only system table query"
  - "Run VACUUM DELETE: YELLOW — reclaims space, resource-intensive, may impact concurrent queries"
  - "Drop unused tables: RED — permanently deletes data"
  - "Elastic resize (add nodes): RED — triggers cluster restart, brief unavailability"
  - "UNLOAD data to S3 before deletion: YELLOW — exports data, requires S3 permissions"
```

## Escalation Conditions
- Cluster serves production analytics
- Disk at 100% blocking all write operations
- Fix requires resize or restore
- Large tables need to be dropped or archived requiring business approval
- Migration from DC2 to RA3 needed for managed storage

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Table names and sizes: database schema information"
    - "Query results: business analytics data"
    - "UNLOAD data in S3: exported business data"
  handling: "UNLOAD data to encrypted S3 buckets. Do not expose table names or sizes externally."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER drop tables without confirming they are truly unused and data is backed up
- NEVER resize a cluster without taking a snapshot first
- NEVER UNLOAD sensitive data to unencrypted S3 buckets

## Phase 3 — Rollback
- If VACUUM DELETE was run: no rollback needed — reclaims already-deleted rows
- If tables were dropped: restore from snapshot (entire cluster) or use table restore for specific tables
- If elastic resize was performed: resize back to original node count (triggers another restart)
- If data was UNLOADed and deleted: COPY data back from S3 if deletion was premature
- If migration to RA3 was started: cannot revert mid-migration — must complete or restore from snapshot

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
  - "NEVER suggest disabling encryption"
