---
title: "D2 — SQL Server Configuration"
description: "Diagnose SQL Server configuration tuning issues on RDS"
status: active
severity: MEDIUM
triggers:
  - "configuration"
  - "sp_configure"
  - "MAXDOP"
  - "cost threshold"
  - "optimize for ad hoc"
  - "query store"
  - "server configuration"
owner: devops-agent
objective: "Identify and resolve SQL Server configuration tuning issues within RDS constraints"
context: "Not all SQL Server configuration options are modifiable on RDS. Configuration is managed via the RDS parameter group, not sp_configure directly. Key tunable parameters include max degree of parallelism, cost threshold for parallelism, optimize for ad hoc workloads, and query store settings."
---

## Phase 1 — Triage

MUST:
- Check current SQL Server configuration:
  ```sql
  SELECT name, value_in_use, minimum, maximum, is_dynamic, is_advanced
  FROM sys.configurations
  WHERE value_in_use != value OR is_advanced = 1
  ORDER BY name;
  ```
- Check RDS parameter group settings: `aws rds describe-db-parameters --db-parameter-group-name <param-group> --source user`
- Identify non-modifiable parameters:
  ```
  aws rds describe-db-parameters --db-parameter-group-name <param-group> \
    --query "Parameters[?IsModifiable==\`false\`].{Name:ParameterName,Value:ParameterValue}" \
    --output table
  ```

SHOULD:
- Check key performance parameters:
  ```sql
  SELECT name, value_in_use FROM sys.configurations WHERE name IN (
    'max degree of parallelism','cost threshold for parallelism',
    'optimize for ad hoc workloads','max server memory (MB)',
    'min server memory (MB)','max worker threads',
    'backup compression default','remote query timeout (s)',
    'clr enabled','cross db ownership chaining'
  ) ORDER BY name;
  ```
- Check Query Store configuration (SQL Server 2016+):
  ```sql
  SELECT actual_state_desc, desired_state_desc, current_storage_size_mb,
         max_storage_size_mb, query_capture_mode_desc
  FROM sys.database_query_store_options;
  ```

MAY:
- Compare with SQL Server best practices for the workload type
- Check trace flags enabled via parameter group

## Phase 2 — Remediate

MUST:
- Modify parameters via RDS parameter group (not sp_configure):
  ```
  aws rds modify-db-parameter-group \
    --db-parameter-group-name <param-group> \
    --parameters '[
      {"ParameterName":"max degree of parallelism","ParameterValue":"4","ApplyMethod":"immediate"},
      {"ParameterName":"cost threshold for parallelism","ParameterValue":"50","ApplyMethod":"immediate"},
      {"ParameterName":"optimize for ad hoc workloads","ParameterValue":"1","ApplyMethod":"immediate"}
    ]'
  ```

SHOULD:
- Key tuning recommendations:
  - `max degree of parallelism`: set to number of cores per NUMA node or 8, whichever is less
  - `cost threshold for parallelism`: increase from default 5 to 25-50 for OLTP workloads
  - `optimize for ad hoc workloads`: enable (1) to reduce plan cache bloat
  - `backup compression default`: enable (1) for faster native backups to S3
- Enable Query Store for plan regression detection:
  ```sql
  ALTER DATABASE [MyDatabase] SET QUERY_STORE = ON;
  ALTER DATABASE [MyDatabase] SET QUERY_STORE (OPERATION_MODE = READ_WRITE, MAX_STORAGE_SIZE_MB = 1024);
  ```

MAY:
- Enable trace flags via parameter group for specific optimizations
- Configure Resource Governor for workload management

## Common Issues

- symptoms: "sp_configure changes not persisting after reboot"
  diagnosis: "sp_configure changes are overridden by RDS parameter group on reboot."
  resolution: "Use RDS parameter group for all configuration changes, not sp_configure."

- symptoms: "Cannot change advanced configuration option"
  diagnosis: "Parameter not modifiable on RDS or requires parameter group change."
  resolution: "Check if parameter is modifiable via describe-db-parameters. Use parameter group."

- symptoms: "CXPACKET waits with default MAXDOP"
  diagnosis: "Default MAXDOP=0 allows unlimited parallelism. Causes excessive parallel plan overhead."
  resolution: "Set max degree of parallelism to cores per NUMA node or 8 via parameter group."

## Safety Ratings
- GREEN: SELECT from sys.configurations, describe-db-parameters, sys.database_query_store_options — read-only inspection
- YELLOW: modify-db-parameter-group (MAXDOP, cost threshold, optimize for ad hoc), ALTER DATABASE SET QUERY_STORE — recoverable changes
- RED: reset-db-parameter-group, delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires parameter group change that needs reboot"
- "Configuration change may affect query plan behavior"
- "Fix involves enabling/disabling Query Store"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, configuration values
- MEDIUM: SQL Server configuration settings, Query Store data, trace flag settings

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix configuration issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest using sp_configure directly — use RDS parameter groups"

## Phase 3 — Rollback
- "Restore from snapshot if configuration change causes issues"
- "Revert parameter group changes and reboot if needed"
- "Document previous configuration values before making changes"

## Output Format

```yaml
root_cause: "sqlserver_config — <specific_cause>"
evidence:
  - type: sql_config
    content: "<sys.configurations output>"
  - type: rds_params
    content: "<parameter group settings>"
severity: MEDIUM
mitigation:
  immediate: "Adjust configuration via RDS parameter group"
  long_term: "Implement configuration baseline and monitoring"
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
