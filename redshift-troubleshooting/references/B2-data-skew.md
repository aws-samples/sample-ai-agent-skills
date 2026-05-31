---
title: "B2 — Data Skew"
description: "Diagnose and resolve data distribution skew across Redshift nodes"
status: active
severity: HIGH
triggers:
  - "data skew"
  - "skew_rows"
  - "uneven distribution"
  - "one node slow"
  - "hot node"
owner: devops-agent
objective: "Identify tables with data skew and fix distribution to balance data across nodes"
context: "Data skew occurs when one compute node slice has significantly more rows than others due to poor distribution key choice. Skewed tables cause uneven query execution — the slowest slice determines overall query time. skew_rows > 4.0 in svv_table_info indicates significant skew."
---

## Phase 1 — Triage

MUST:
- Check table skew: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT schema, \"table\", diststyle, skew_rows, skew_sortkey1, size FROM svv_table_info WHERE skew_rows > 2.0 ORDER BY skew_rows DESC LIMIT 20"`
- Check per-slice row distribution: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT trim(name) AS tablename, slice, num_values FROM svv_diskusage WHERE name = '<table>' ORDER BY slice"`
- Check distribution style of skewed tables: `aws redshift-data execute-statement --cluster-identifier <cluster-id> --database <db> --db-user <user> --sql "SELECT tablename, diststyle FROM pg_catalog.svv_table_info WHERE \"table\" = '<table>'"`
- Check if skew correlates with slow queries on those tables

SHOULD:
- Analyze distribution key cardinality: `SELECT <dist_key>, COUNT(*) FROM <table> GROUP BY 1 ORDER BY 2 DESC LIMIT 20`
- Check for NULL values in distribution key (NULLs all go to one slice)
- Review join patterns to determine optimal distribution key

MAY:
- Compare query performance before and after distribution change
- Check if AUTO distribution chose a suboptimal key
- Analyze query plans for redistribution steps (DS_DIST_*)

## Phase 2 — Remediate

MUST:
- If skew caused by low-cardinality distribution key: change to a high-cardinality column
- If skew caused by NULLs in distribution key: choose a non-nullable column or use EVEN distribution
- If small table (< 1M rows): consider ALL distribution for join performance

SHOULD:
- Choose distribution key that matches the most common join column
- Ensure distribution key has high cardinality and even value distribution
- Test with EVEN distribution as a baseline before choosing KEY distribution

MAY:
- Use AUTO distribution and let Redshift choose
- Create a deep copy of the table with new distribution: `CREATE TABLE new_table (LIKE old_table) DISTSTYLE KEY DISTKEY(column); INSERT INTO new_table SELECT * FROM old_table;`
- Monitor skew after changes

## Common Issues

- symptoms: "skew_rows = 15.0 on a large fact table"
  diagnosis: "Distribution key has low cardinality or many NULL values."
  resolution: "Change distribution key to a high-cardinality column (e.g., customer_id instead of status)."

- symptoms: "Queries on evenly distributed table still show one slice much slower"
  diagnosis: "Sort key skew — data is sorted but filter predicates hit one slice disproportionately."
  resolution: "This is filter skew, not distribution skew. Consider interleaved sort key or partition the data."

## Output Format

```yaml
root_cause: "Data skew — <specific_cause>"
evidence:
  - type: system_table
    content: "<svv_table_info skew_rows values>"
severity: HIGH
mitigation:
  immediate: "Redistribute table with better distribution key"
  long_term: "Review distribution keys for all large tables, monitor skew_rows regularly"
```


## Safety Ratings
```
safety_ratings:
  - "Query svv_table_info for skew: GREEN — read-only system table query"
  - "Check per-slice distribution: GREEN — read-only system table query"
  - "Analyze distribution key cardinality: GREEN — read-only query"
  - "Change distribution style via deep copy: RED — creates new table, requires significant I/O and storage"
  - "Use AUTO distribution: YELLOW — lets Redshift choose, may change distribution unexpectedly"
```

## Escalation Conditions
- Cluster serves production analytics
- Severe data skew (skew_rows > 10) causing query performance degradation
- Fix requires distribution key change on large production tables
- Deep copy requires sufficient free disk space (2x table size)
- Distribution change affects join performance across multiple queries

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Distribution key values: may reveal business data patterns"
    - "Table schemas and column names: database structure"
    - "Query results: business analytics data"
  handling: "Do not expose distribution key analysis or table structure externally."
```

## Prohibited Actions
- NEVER suggest deleting a cluster without snapshot
- NEVER suggest VACUUM FULL on production during peak hours
- NEVER change distribution keys on production tables without a maintenance window
- NEVER perform deep copy without verifying sufficient free disk space
- NEVER use EVEN distribution on tables frequently joined on a specific column without understanding the performance impact

## Phase 3 — Rollback
- If deep copy was performed: rename original table back and drop the new table (if original was preserved)
- If AUTO distribution was set: change back to explicit KEY or EVEN distribution
- If distribution key was changed: perform another deep copy with the original distribution key
- Take a snapshot before any distribution changes as a safety net

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
