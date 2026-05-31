---
title: "H1 — TDE Issues"
description: "Diagnose SQL Server Transparent Data Encryption (TDE) issues on RDS SQL Server"
status: active
severity: HIGH
triggers:
  - "TDE"
  - "Transparent Data Encryption"
  - "encryption key"
  - "encryption at rest"
  - "database encryption"
  - "KMS"
owner: devops-agent
objective: "Resolve TDE configuration and operational issues on RDS SQL Server Enterprise Edition"
context: "TDE on RDS SQL Server requires Enterprise Edition. TDE is configured via RDS option groups and uses AWS KMS for key management. TDE encrypts data at rest at the database level. The TDE certificate is managed by RDS. Express, Web, and Standard editions do NOT support TDE."
---

## Phase 1 — Triage

MUST:
- Check if TDE option is configured: `aws rds describe-option-groups --option-group-name <option-group> --query "OptionGroupsList[0].Options[?OptionName=='TRANSPARENT_DATA_ENCRYPTION']"`
- Check SQL Server edition (must be Enterprise): `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].Engine'`
- Check TDE status in SQL Server:
  ```sql
  SELECT db.name, db.is_encrypted,
         dek.encryption_state, dek.key_algorithm, dek.key_length
  FROM sys.databases db
  LEFT JOIN sys.dm_database_encryption_keys dek ON db.database_id = dek.database_id
  ORDER BY db.name;
  ```
  Encryption states: 0=No key, 1=Unencrypted, 2=Encryption in progress, 3=Encrypted, 4=Key change in progress, 5=Decryption in progress
- Check KMS key status: `aws kms describe-key --key-id <key-id>`

SHOULD:
- Verify option group is associated with the instance: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].OptionGroupMemberships'`
- Check for TDE-related errors in error log:
  ```
  aws rds download-db-log-file-portion --db-instance-identifier <instance-id> --log-file-name error/ERRORLOG
  ```
- Check TDE certificate:
  ```sql
  SELECT name, pvt_key_encryption_type_desc, start_date, expiry_date
  FROM sys.certificates WHERE name LIKE 'RDSTDECertificate%';
  ```

MAY:
- Check backup encryption status (TDE-encrypted databases produce encrypted backups)
- Verify TDE performance impact

## Phase 2 — Remediate

MUST:
- Enable TDE via option group:
  ```
  aws rds add-option-to-option-group \
    --option-group-name <option-group> \
    --options "OptionName=TRANSPARENT_DATA_ENCRYPTION"
  ```
- Associate option group with instance:
  ```
  aws rds modify-db-instance \
    --db-instance-identifier <instance-id> \
    --option-group-name <option-group> \
    --apply-immediately
  ```
- Enable TDE on a database:
  ```sql
  -- After TDE option is active
  USE [master];
  CREATE DATABASE ENCRYPTION KEY
    WITH ALGORITHM = AES_256
    ENCRYPTION BY SERVER CERTIFICATE [RDSTDECertificate];
  ALTER DATABASE [MyDatabase] SET ENCRYPTION ON;
  ```

SHOULD:
- Monitor encryption progress:
  ```sql
  SELECT db_name(database_id) AS db_name, encryption_state, percent_complete
  FROM sys.dm_database_encryption_keys;
  ```
- Note: enabling TDE on any database also encrypts tempdb automatically

MAY:
- Rotate TDE certificate (managed by RDS via option group)
- Monitor CPU impact after enabling TDE

## Common Issues

- symptoms: "Cannot enable TDE — option not available"
  diagnosis: "TDE requires Enterprise Edition. Not available on Express, Web, or Standard."
  resolution: "Migrate to Enterprise Edition to use TDE. Or use RDS storage encryption (EBS encryption) as alternative."

- symptoms: "TDE encryption taking very long"
  diagnosis: "Large database encryption is I/O intensive. Progress tracked via percent_complete."
  resolution: "Wait for completion. Monitor via sys.dm_database_encryption_keys. Avoid during peak hours."

- symptoms: "Performance impact after enabling TDE"
  diagnosis: "TDE adds CPU overhead for encryption/decryption of data pages."
  resolution: "Monitor CPU after enabling. Typically 3-5% overhead. Scale instance if needed."

## Safety Ratings
- GREEN: describe-option-groups, SELECT from sys.dm_database_encryption_keys/sys.certificates — read-only inspection
- YELLOW: add-option-to-option-group (TDE), ALTER DATABASE SET ENCRYPTION ON — recoverable but adds CPU overhead
- RED: delete-db-instance, disabling TDE — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "TDE certificate issues — encrypted data inaccessible"
- "Fix involves modifying encryption settings"
- "KMS key disabled or pending deletion"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: TDE certificates, KMS key IDs, encryption configuration (core security infrastructure)
- HIGH: database credentials, connection strings
- MEDIUM: encrypted database inventory, option group configuration

## Prohibited Actions
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest deleting a database instance to fix TDE issues — use snapshots"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest deleting TDE certificates without understanding impact"

## Phase 3 — Rollback
- "Restore from snapshot if TDE configuration change causes data access issues"
- "If KMS key was disabled, re-enable immediately"

## Output Format

```yaml
root_cause: "tde — <specific_cause>"
evidence:
  - type: option_group
    content: "<TDE option configuration>"
  - type: encryption_status
    content: "<sys.dm_database_encryption_keys output>"
severity: HIGH
mitigation:
  immediate: "Fix TDE configuration or wait for encryption to complete"
  long_term: "Implement TDE monitoring and certificate rotation schedule"
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
