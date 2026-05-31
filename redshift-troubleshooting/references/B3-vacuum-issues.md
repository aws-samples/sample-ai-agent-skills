---
title: "B3 — VACUUM Issues"
description: "Diagnose and resolve VACUUM operation problems in Redshift"
status: active
severity: MEDIUM
triggers:
  - "VACUUM"
  - "unsorted rows"
  - "deleted rows"
  - "ghost rows"
  - "table bloat"
  - "VACUUM taking long"
owner: devops-agent
objective: "Identify tables needing VACUUM and resolve VACUUM operation issues"
context: "VACUUM reclaims space from deleted rows (VACUUM DELETE) and re-sorts unsorted rows (VACUUM SORT). Auto-VACUUM handles DELETE reclamation in the background. Manual VACUUM SORT may be needed for tables with heavy insert activity. VACUUM FULL does both."
---

## Phase 1 — Triage

MUST:
- Check tables needing VACUUM: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT schema, \"table\", size, unsorted, empty, tbl_rows FROM svv_table_info WHERE unsorted > 5 OR empty > 20 ORDER BY size DESC LIMIT 20"`
- Check running VACUUM operations: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT * FROM svv_vacuum_progress"`
- Check auto-VACUUM status: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT * FROM svl_vacuum WHERE table_name = '<table>' ORDER BY starttime DESC LIMIT 5"`
- Check disk space (VACUUM needs working space): `aws cloudwatch get-metric-statistics --namespace AWS/Redshift --metric-name PercentageDiskSpaceUsed --dimensions Name=ClusterIdentifier,Value=<cluster-id> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 300 --statistics Maximum`

SHOULD:
- Check VACUUM history for failures: `SELECT * FROM stl_vacuum WHERE status != 'Complete' ORDER BY starttime DESC LIMIT 10`
- Verify sort key type (compound vs interleaved — interleaved VACUUM is much slower)
- Check if table has been recently loaded with large batch

MAY:
- Check auto-VACUUM threshold settings in parameter group
- Review VACUUM BOOST option for faster execution
- Check if concurrent queries are blocking VACUUM

## Phase 2 — Remediate

MUST:
- If high unsorted percentage: run `VACUUM SORT <table>` or `VACUUM FULL <table>`
- If high empty percentage: run `VACUUM DELETE <table>`
- If VACUUM fails due to disk space: free space first, then VACUUM smaller tables
- If VACUUM takes too long: use `VACUUM BOOST <table>` for faster execution (uses more resources)

SHOULD:
- Schedule VACUUM during off-peak hours
- Run VACUUM on largest tables first
- Use VACUUM DELETE ONLY if sort is not needed (faster)

MAY:
- Consider deep copy (CREATE TABLE AS) for heavily fragmented tables
- Adjust auto-VACUUM threshold in parameter group
- Implement automated VACUUM scheduling via Lambda or Step Functions

## Common Issues

- symptoms: "Table has 40% unsorted rows, queries are slow"
  diagnosis: "Large data loads added unsorted rows. Sort key scans are inefficient."
  resolution: "Run VACUUM SORT <table>. For very large tables, use VACUUM BOOST."

- symptoms: "VACUUM runs for hours and doesn't complete"
  diagnosis: "Very large table with interleaved sort key. Interleaved VACUUM is O(n) on table size."
  resolution: "Use VACUUM REINDEX for interleaved sort keys. Consider switching to compound sort key."

- symptoms: "Disk space not reclaimed after DELETE"
  diagnosis: "Deleted rows marked as ghost rows but not reclaimed. Auto-VACUUM may be delayed."
  resolution: "Run VACUUM DELETE <table> manually. Check auto-VACUUM is enabled."

## Output Format

```yaml
root_cause: "VACUUM issue — <specific_cause>"
evidence:
  - type: system_table
    content: "<svv_table_info unsorted and empty percentages>"
severity: MEDIUM
mitigation:
  immediate: "Run appropriate VACUUM command on affected tables"
  long_term: "Schedule regular VACUUM, monitor unsorted/empty percentages, consider deep copy for heavily fragmented tables"
```


## Safety Ratings
```
safety_ratings:
  - "Query svv_table_info for unsorted/empty: GREEN — read-only system table query"
  - "Check svv_vacuum_progress: GREEN — read-only system table query"
  - "Run VACUUM DELETE: YELLOW — reclaims space, resource-intensive"
  - "Run VACUUM SORT: YELLOW — re-sorts data, resource-intensive, may impact concurrent queries"
  - "Run VACUUM FULL: RED — combines DELETE and SORT, very resource-intensive, long-running"
  - "Run VACUUM BOOST: YELLOW — uses all available resources, may significantly impact concurrent queries"
  - "Deep copy for heavily fragmented tables: RED — requires significant I/O and storage"
```

## Escalation Conditions
- Cluster serves production analytics
- Tables with >50% unsorted rows causing severe query degradation
- VACUUM operations failing due to insufficient disk space
- Interleaved sort key VACUUM taking excessively long
- VACUUM FULL needed on large production tables during business hours

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Table names and fragmentation metrics: database health indicators"
    - "VACUUM history: operational patterns"
  handling: "Do not expose table fragmentation details externally."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER run VACUUM BOOST during peak query hours — it consumes all available resources
- NEVER run VACUUM on a cluster with <25% free disk space without freeing space first
- NEVER interrupt a running VACUUM REINDEX — it may leave the table in an inconsistent sort state

## Phase 3 — Rollback
- VACUUM DELETE: no rollback needed — only reclaims already-deleted rows
- VACUUM SORT: no rollback needed — data is re-sorted in place, non-destructive
- VACUUM FULL: no rollback needed — combines DELETE and SORT
- If deep copy was performed: rename original table back if preserved
- If auto-VACUUM threshold was changed: revert parameter group to previous threshold value

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
