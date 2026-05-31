---
title: "G3 — Collation Problems"
description: "Diagnose SQL Server collation issues on RDS SQL Server"
status: active
severity: MEDIUM
triggers:
  - "collation"
  - "collation conflict"
  - "SQL_Latin1_General"
  - "Latin1_General"
  - "case sensitive"
  - "sort order"
  - "Cannot resolve collation conflict"
owner: devops-agent
objective: "Resolve collation-related issues on RDS SQL Server"
context: "RDS SQL Server server-level collation is set at instance creation and CANNOT be changed afterward. Default is SQL_Latin1_General_CP1_CI_AS. Database-level collation can be set at database creation. Collation conflicts occur when joining columns with different collations. tempdb inherits the server collation."
---

## Phase 1 — Triage

MUST:
- Check server collation:
  ```sql
  SELECT SERVERPROPERTY('Collation') AS server_collation;
  ```
- Check database collation:
  ```sql
  SELECT name, collation_name FROM sys.databases ORDER BY name;
  ```
- Check column collations for affected tables:
  ```sql
  SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME, COLLATION_NAME
  FROM INFORMATION_SCHEMA.COLUMNS
  WHERE TABLE_NAME = '<table>' AND COLLATION_NAME IS NOT NULL;
  ```
- Identify the specific collation conflict error and the columns involved

SHOULD:
- Check tempdb collation (always matches server collation):
  ```sql
  SELECT DATABASEPROPERTYEX('tempdb', 'Collation') AS tempdb_collation;
  ```
- Compare collations between databases involved in cross-database queries
- Check for implicit collation conversions in query plans

MAY:
- List all distinct collations in use across the instance:
  ```sql
  SELECT DISTINCT collation_name, COUNT(*) AS column_count
  FROM INFORMATION_SCHEMA.COLUMNS
  WHERE COLLATION_NAME IS NOT NULL
  GROUP BY collation_name ORDER BY column_count DESC;
  ```

## Phase 2 — Remediate

MUST:
- Server collation CANNOT be changed on existing RDS instance
- For server collation change: create new instance with correct collation and migrate data
  ```
  aws rds create-db-instance \
    --db-instance-identifier <new-instance> \
    --engine sqlserver-se \
    --character-set-name Latin1_General_CI_AS \
    ...
  ```
- For collation conflicts in queries: use COLLATE clause
  ```sql
  SELECT a.col1, b.col2
  FROM TableA a JOIN TableB b
    ON a.col1 COLLATE SQL_Latin1_General_CP1_CI_AS = b.col2 COLLATE SQL_Latin1_General_CP1_CI_AS;
  ```

SHOULD:
- For database-level collation: set at database creation
  ```sql
  CREATE DATABASE [MyDatabase] COLLATE Latin1_General_CI_AS;
  ```
- For column-level collation fix:
  ```sql
  ALTER TABLE [MyTable] ALTER COLUMN [MyColumn] NVARCHAR(100) COLLATE Latin1_General_CI_AS;
  ```
- Standardize collation across all databases and columns where possible

MAY:
- Use COLLATE DATABASE_DEFAULT in temp table definitions to avoid tempdb collation conflicts:
  ```sql
  CREATE TABLE #temp (col1 NVARCHAR(100) COLLATE DATABASE_DEFAULT);
  ```
- Document collation standards for the organization

## Common Issues

- symptoms: "Cannot resolve the collation conflict between collations in the equal to operation"
  diagnosis: "Joining or comparing columns with different collations."
  resolution: "Add COLLATE clause to the query. Or alter column collation to match."

- symptoms: "Temp table collation mismatch with user database"
  diagnosis: "tempdb uses server collation. If database collation differs, temp table joins fail."
  resolution: "Use COLLATE DATABASE_DEFAULT when creating temp table columns."

- symptoms: "Need to change server collation but cannot on RDS"
  diagnosis: "Server collation is immutable after instance creation."
  resolution: "Create new RDS instance with desired collation. Migrate data using native backup/restore."

## Safety Ratings
- GREEN: SELECT from sys.fn_helpcollations()/SERVERPROPERTY('Collation')/DATABASEPROPERTYEX — read-only inspection
- YELLOW: ALTER DATABASE COLLATE (requires single-user mode) — recoverable but high-impact
- RED: delete-db-instance, creating new instance with different collation — high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Collation change requires single-user mode (causes downtime)"
- "Server-level collation cannot be changed — requires new instance"
- "Collation mismatch causing query failures"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings
- MEDIUM: collation configuration, column-level collation details

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix collation issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest changing server collation on existing RDS instance — it is immutable"

## Phase 3 — Rollback
- "Revert database collation changes if they cause query failures"
- "Restore from snapshot if collation change causes data issues"

## Output Format

```yaml
root_cause: "collation — <specific_cause>"
evidence:
  - type: server_collation
    content: "<server collation>"
  - type: database_collation
    content: "<database collations>"
  - type: column_collation
    content: "<affected column collations>"
severity: MEDIUM
mitigation:
  immediate: "Use COLLATE clause in queries or alter column collation"
  long_term: "Standardize collation across all databases"
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
