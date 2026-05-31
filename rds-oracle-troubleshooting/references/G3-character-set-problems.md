---
title: "G3 — Character Set Problems"
description: "Diagnose Oracle character set and NLS issues on RDS Oracle"
status: active
severity: MEDIUM
triggers:
  - "character set"
  - "NLS"
  - "AL32UTF8"
  - "WE8ISO8859P1"
  - "garbled text"
  - "encoding"
  - "ORA-29275"
  - "national character set"
owner: devops-agent
objective: "Resolve character set and NLS-related issues on RDS Oracle"
context: "RDS Oracle character set is set at instance creation and CANNOT be changed afterward. Default is AL32UTF8. National character set default is AL16UTF16. NLS session parameters (NLS_LANGUAGE, NLS_TERRITORY, NLS_DATE_FORMAT) can be set per session or via parameter group. Character set mismatches between client and server cause garbled text."
---

## Phase 1 — Triage

MUST:
- Check database character set:
  ```sql
  SELECT value FROM nls_database_parameters WHERE parameter = 'NLS_CHARACTERSET';
  SELECT value FROM nls_database_parameters WHERE parameter = 'NLS_NCHAR_CHARACTERSET';
  ```
- Check session NLS settings:
  ```sql
  SELECT parameter, value FROM nls_session_parameters ORDER BY parameter;
  ```
- Check client NLS_LANG environment variable (client-side)
- Identify the specific characters or data causing issues

SHOULD:
- Check for character set conversion issues:
  ```sql
  SELECT column_name, data_type, character_set_name
  FROM all_tab_columns WHERE table_name = '<TABLE>' AND owner = '<OWNER>';
  ```
- Verify client character set matches or is compatible with database:
  ```sql
  SELECT * FROM v$nls_parameters WHERE parameter IN ('NLS_CHARACTERSET','NLS_LANGUAGE','NLS_TERRITORY');
  ```
- Check for data with characters outside the target character set:
  ```sql
  SELECT <column>, DUMP(<column>) FROM <table>
  WHERE <column> IS NOT NULL AND ROWNUM <= 10;
  ```

MAY:
- Check for NCHAR/NVARCHAR2 columns (use national character set)
- Verify JDBC connection string NLS settings

## Phase 2 — Remediate

MUST:
- Character set CANNOT be changed on existing RDS instance
- For character set change: create new instance with correct character set and migrate data
  ```
  aws rds create-db-instance \
    --db-instance-identifier <new-instance> \
    --engine oracle-ee \
    --character-set-name WE8ISO8859P1 \
    ...
  ```
- For NLS session parameters: set via parameter group or session-level
  ```
  aws rds modify-db-parameter-group \
    --db-parameter-group-name <param-group> \
    --parameters "ParameterName=nls_date_format,ParameterValue=YYYY-MM-DD HH24:MI:SS,ApplyMethod=immediate"
  ```

SHOULD:
- Set client NLS_LANG to match database character set
- For JDBC: set connection properties
  ```
  jdbc:oracle:thin:@//<endpoint>:1521/<service>
  # With NLS properties
  props.setProperty("oracle.jdbc.defaultNChar", "true");
  ```
- Use AL32UTF8 for new instances (supports all Unicode characters)

MAY:
- Use CONVERT function for data transformation:
  ```sql
  SELECT CONVERT(<column>, 'AL32UTF8', 'WE8ISO8859P1') FROM <table>;
  ```
- Implement application-level character encoding handling

## Common Issues

- symptoms: "Garbled characters (mojibake) in query results"
  diagnosis: "Client NLS_LANG doesn't match database character set."
  resolution: "Set NLS_LANG on client to match database (e.g., AMERICAN_AMERICA.AL32UTF8)."

- symptoms: "ORA-12899: value too large for column after migration"
  diagnosis: "Character set change from single-byte (WE8ISO8859P1) to multi-byte (AL32UTF8) increases byte length."
  resolution: "Increase column size or use CHAR length semantics: ALTER TABLE ... MODIFY (<col> VARCHAR2(100 CHAR))."

- symptoms: "Need to change character set but cannot on RDS"
  diagnosis: "Character set is immutable after instance creation."
  resolution: "Create new instance with desired character set. Migrate data using Data Pump with REMAP_DATA or character set conversion."

## Safety Ratings
- GREEN: SELECT from nls_database_parameters/nls_session_parameters/v$nls_parameters/all_tab_columns — read-only inspection
- YELLOW: modify-db-parameter-group (NLS session parameters), ALTER SESSION SET NLS_* — recoverable changes
- RED: create-db-instance (new instance with different character set requires data migration), delete-db-instance — high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Character set change requires creating new instance and migrating data"
- "Garbled text affecting application data integrity"
- "Fix requires application-wide NLS_LANG configuration changes"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, data samples from DUMP() function
- MEDIUM: character set configuration, NLS parameters, column metadata

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix character set issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"
- "NEVER suggest changing character set on existing RDS instance — it is immutable"

## Phase 3 — Rollback
- "Revert NLS session parameter changes if they cause application issues"
- "Revert parameter group NLS changes and reboot if needed"
- "If data migration to new character set instance fails, keep original instance running"

## Output Format

```yaml
root_cause: "character_set — <specific_cause>"
evidence:
  - type: db_charset
    content: "<NLS_CHARACTERSET value>"
  - type: session_nls
    content: "<NLS session parameters>"
severity: MEDIUM
mitigation:
  immediate: "Fix client NLS_LANG or session NLS parameters"
  long_term: "Standardize on AL32UTF8 for new instances"
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
