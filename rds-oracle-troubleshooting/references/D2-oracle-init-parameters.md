---
title: "D2 — Oracle Init Parameters"
description: "Diagnose Oracle initialization parameter tuning issues on RDS"
status: active
severity: MEDIUM
triggers:
  - "init parameter"
  - "init.ora"
  - "spfile"
  - "Oracle parameter"
  - "sga_target"
  - "pga_aggregate_target"
  - "open_cursors"
  - "db_files"
  - "cursor_sharing"
owner: devops-agent
objective: "Identify and resolve Oracle init parameter tuning issues within RDS constraints"
context: "Not all Oracle init parameters are modifiable on RDS. RDS manages parameters like db_block_size, compatible, and control_files. Modifiable parameters are set via the RDS parameter group. Some use formula variables like {DBInstanceClassMemory}. memory_target (AMM) is not supported — use ASMM."
---

## Phase 1 — Triage

MUST:
- Check current Oracle parameter values:
  ```sql
  SELECT name, value, isdefault, ismodified, description
  FROM v$parameter WHERE isdefault = 'FALSE' ORDER BY name;
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
  SELECT name, value FROM v$parameter WHERE name IN (
    'sga_target','sga_max_size','pga_aggregate_target',
    'processes','sessions','open_cursors','session_cached_cursors',
    'cursor_sharing','optimizer_mode','db_file_multiblock_read_count',
    'sort_area_size','hash_area_size','db_files','undo_retention'
  ) ORDER BY name;
  ```
- Check hidden parameters (if needed):
  ```sql
  SELECT ksppinm AS name, ksppstvl AS value
  FROM x$ksppi a JOIN x$ksppcv b ON a.indx = b.indx
  WHERE ksppinm LIKE '\_%' ESCAPE '\' AND ksppstvl IS NOT NULL;
  ```

MAY:
- Compare with Oracle best practices for the workload type
- Check parameter change history in RDS events

## Phase 2 — Remediate

MUST:
- Modify parameters via RDS parameter group (not ALTER SYSTEM):
  ```
  aws rds modify-db-parameter-group \
    --db-parameter-group-name <param-group> \
    --parameters '[
      {"ParameterName":"open_cursors","ParameterValue":"500","ApplyMethod":"immediate"},
      {"ParameterName":"session_cached_cursors","ParameterValue":"200","ApplyMethod":"immediate"}
    ]'
  ```
- For memory parameters, use formulas:
  - `sga_target`: `{DBInstanceClassMemory*3/4}` (default)
  - `pga_aggregate_target`: `{DBInstanceClassMemory/8}` (typical)
- Static parameters (processes, db_files) require reboot after change

SHOULD:
- Key tuning recommendations:
  - `processes`: increase for high-connection workloads (sessions = 1.5 * processes + 22)
  - `open_cursors`: 300-500 for typical OLTP
  - `session_cached_cursors`: 100-200 to reduce soft parsing
  - `cursor_sharing`: leave as EXACT unless hard parsing is severe (then FORCE)
  - `undo_retention`: increase for long-running queries (ORA-01555)
- Test parameter changes in non-production first

MAY:
- Use Oracle advisors to guide tuning:
  ```sql
  SELECT * FROM v$sga_target_advice;
  SELECT * FROM v$pga_target_advice;
  SELECT * FROM v$shared_pool_advice;
  ```

## Common Issues

- symptoms: "Cannot set memory_target on RDS"
  diagnosis: "Automatic Memory Management (AMM) is not supported on RDS."
  resolution: "Use ASMM: set sga_target and pga_aggregate_target separately."

- symptoms: "ALTER SYSTEM SET parameter fails"
  diagnosis: "Parameters must be set via RDS parameter group, not ALTER SYSTEM."
  resolution: "Use modify-db-parameter-group CLI command."

- symptoms: "ORA-00020: maximum number of processes exceeded"
  diagnosis: "processes parameter too low for the workload."
  resolution: "Increase processes in parameter group. Requires reboot (static parameter)."

## Safety Ratings
- GREEN: SELECT from v$parameter, describe-db-parameters, Oracle advisor views (v$sga_target_advice, v$pga_target_advice) — read-only inspection
- YELLOW: modify-db-parameter-group (open_cursors, session_cached_cursors, undo_retention) — recoverable but may require reboot for static params
- RED: reset-db-parameter-group, delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires parameter group change that needs reboot"
- "Fix involves modifying SGA/PGA memory allocation"
- "Parameter change may affect application behavior"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, parameter values (may contain security-sensitive settings)
- HIGH: hidden parameter values (x$ksppi) — reveal internal Oracle configuration
- MEDIUM: Oracle advisor output, parameter group settings

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix parameter issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"
- "NEVER suggest setting memory_target on RDS Oracle — it is not supported"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "Revert parameter group changes and reboot if needed"
- "Document previous parameter values before making changes to enable rollback"
- "If SGA/PGA changes cause ORA-04031, revert and reboot"

## Output Format

```yaml
root_cause: "init_parameter — <specific_cause>"
evidence:
  - type: oracle_params
    content: "<v$parameter output>"
  - type: rds_params
    content: "<parameter group settings>"
severity: MEDIUM
mitigation:
  immediate: "Adjust parameter via RDS parameter group"
  long_term: "Implement parameter baseline and monitoring"
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
