---
title: "A1 — Slow Queries"
description: "Diagnose and resolve slow query performance in Redshift"
status: active
severity: HIGH
triggers:
  - "slow query"
  - "query taking long"
  - "query performance"
  - "query timeout"
  - "high query duration"
owner: devops-agent
objective: "Identify the root cause of slow queries and optimize performance"
context: "Redshift query performance depends on distribution keys, sort keys, WLM configuration, table statistics, data skew, and query plan efficiency. Slow queries can be caused by full table scans, redistribution steps, disk-based operations, or resource contention."
---

## Phase 1 — Triage

MUST:
- Check recent slow queries: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT query, elapsed, substring, label FROM svl_qlog WHERE elapsed > 60000000 ORDER BY starttime DESC LIMIT 20"`
- Check query execution steps: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT query, seg, step, label, rows, bytes, is_diskbased FROM svl_query_summary WHERE query = <query-id> ORDER BY seg, step"`
- Check CPU utilization: `aws cloudwatch get-metric-statistics --namespace AWS/Redshift --metric-name CPUUtilization --dimensions Name=ClusterIdentifier,Value=<cluster-id> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 300 --statistics Average`
- Check WLM queue wait time: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT query, service_class, total_queue_time, total_exec_time FROM svl_query_metrics WHERE query = <query-id>"`

SHOULD:
- Check for disk-based operations: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT query, segment, step, label, is_diskbased, workmem FROM svl_query_summary WHERE query = <query-id> AND is_diskbased = 't'"`
- Check table statistics freshness: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT schema, \"table\", stats_off FROM svv_table_info WHERE stats_off > 10 ORDER BY stats_off DESC"`
- Check for nested loop joins (performance killer): look for "NL" in query plan
- Verify EXPLAIN plan: `EXPLAIN <slow-query>`

MAY:
- Check compile time: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT * FROM svl_compile WHERE query = <query-id>"`
- Check network redistribution steps in query plan
- Review query alerts: `SELECT * FROM stl_alert_event_log WHERE query = <query-id>`

## Phase 2 — Remediate

MUST:
- If disk-based operations: increase WLM memory allocation or reduce query concurrency
- If stale statistics: run `ANALYZE <table>` on affected tables
- If full table scan on sorted column: verify sort key usage and VACUUM status
- If redistribution steps: review distribution keys for joined tables

SHOULD:
- Run ANALYZE on tables with stats_off > 10%
- Run VACUUM SORT on tables with unsorted > 5%
- Consider changing distribution style for frequently joined tables
- Use Auto WLM instead of manual WLM for dynamic memory allocation

MAY:
- Add result caching for repeated queries
- Create materialized views for complex aggregations
- Consider short query acceleration (SQA) for sub-second queries

## Common Issues

- symptoms: "Query runs for 30 minutes, EXPLAIN shows nested loop join"
  diagnosis: "Nested loop join caused by missing or incorrect join condition, or small table broadcast."
  resolution: "Fix join condition. If intentional, ensure smaller table uses ALL distribution."

- symptoms: "Query is disk-based (is_diskbased = true in svl_query_summary)"
  diagnosis: "Insufficient memory allocated to the query — data spills to disk."
  resolution: "Increase WLM memory percentage or reduce concurrency. Switch to Auto WLM."

- symptoms: "Query plan shows DS_DIST_ALL_INNER or DS_BCAST_INNER redistribution"
  diagnosis: "Tables being joined have incompatible distribution keys."
  resolution: "Set distribution key to the join column on both tables, or use ALL distribution for small dimension tables."

## Output Format

```yaml
root_cause: "Slow query — <specific_cause>"
evidence:
  - type: system_table
    content: "<svl_query_summary or svl_qlog output>"
  - type: cloudwatch_metric
    content: "<CPU or disk metrics>"
severity: HIGH
mitigation:
  immediate: "Apply the identified optimization (ANALYZE, VACUUM, WLM change, or distribution key fix)"
  long_term: "Implement regular ANALYZE/VACUUM schedules, review distribution keys for key tables"
```


## Safety Ratings
```
safety_ratings:
  - "Query svl_qlog and svl_query_summary: GREEN — read-only system table queries"
  - "Check CloudWatch metrics: GREEN — read-only monitoring query"
  - "Run EXPLAIN on slow query: GREEN — read-only query plan analysis"
  - "Run ANALYZE on tables: YELLOW — updates statistics, brief table lock"
  - "Run VACUUM SORT: YELLOW — re-sorts data, resource-intensive, may impact concurrent queries"
  - "Change distribution style: RED — requires table rebuild (deep copy), significant I/O and downtime"
  - "Modify WLM memory allocation: YELLOW — parameter group change may require cluster restart"
```

## Escalation Conditions
- Cluster serves production analytics
- Slow queries impacting SLA for business-critical dashboards
- Fix requires resize or restore
- Distribution key change needed on large production tables
- WLM configuration change requires cluster restart

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Query text in svl_qlog: may contain sensitive SQL with business data"
    - "User credentials: database usernames in query logs"
    - "Query results: business analytics data"
  handling: "Mask query text containing PII. Do not expose database usernames externally."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER change distribution keys on large tables without a maintenance window and deep copy plan
- NEVER terminate user sessions without confirming the impact on running transactions

## Phase 3 — Rollback
- If ANALYZE was run: no rollback needed — statistics updates are non-destructive
- If VACUUM SORT was run: cannot be undone, but is non-destructive (data is re-sorted in place)
- If WLM memory allocation was changed: revert parameter group to previous WLM JSON configuration
- If distribution style was changed via deep copy: rename original table back and drop the new table
- If sort key was changed: restore from snapshot or perform another deep copy with original sort key

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
