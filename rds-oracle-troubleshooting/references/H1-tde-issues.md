---
title: "H1 — TDE Issues"
description: "Diagnose Oracle Transparent Data Encryption (TDE) issues on RDS Oracle"
status: active
severity: HIGH
triggers:
  - "TDE"
  - "Transparent Data Encryption"
  - "encryption key"
  - "ORA-28365"
  - "ORA-28374"
  - "wallet"
  - "encryption at rest"
  - "CloudHSM"
owner: devops-agent
objective: "Resolve TDE configuration and operational issues on RDS Oracle Enterprise Edition"
context: "TDE on RDS Oracle requires Enterprise Edition with the Oracle Advanced Security option. TDE is configured via RDS option groups, not SQL commands. Key management uses AWS KMS (default) or AWS CloudHSM. TDE encrypts data at the tablespace or column level. The TDE wallet is managed by RDS."
---

## Phase 1 — Triage

MUST:
- Check if TDE option is configured: `aws rds describe-option-groups --option-group-name <option-group> --query "OptionGroupsList[0].Options[?OptionName=='TDE']"`
- Check Oracle edition (must be EE): `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].Engine'`
- Check TDE status in Oracle:
  ```sql
  SELECT * FROM v$encryption_wallet;
  SELECT tablespace_name, encrypted FROM dba_tablespaces;
  SELECT owner, table_name, column_name, encryption_alg FROM dba_encrypted_columns;
  ```
- Check KMS key status (if using KMS): `aws kms describe-key --key-id <key-id>`

SHOULD:
- Verify option group is associated with the instance: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].OptionGroupMemberships'`
- Check for TDE-related errors in alert log: `aws rds download-db-log-file-portion --db-instance-identifier <instance-id> --log-file-name trace/alert_<SID>.log`
- Verify CloudHSM cluster status (if using CloudHSM): `aws cloudhsmv2 describe-clusters`

MAY:
- Check TDE master key rotation history
- Verify backup encryption status

## Phase 2 — Remediate

MUST:
- Enable TDE via option group:
  ```
  aws rds add-option-to-option-group \
    --option-group-name <option-group> \
    --options "OptionName=TDE"
  ```
- Associate option group with instance:
  ```
  aws rds modify-db-instance \
    --db-instance-identifier <instance-id> \
    --option-group-name <option-group> \
    --apply-immediately
  ```
- Encrypt tablespace:
  ```sql
  ALTER TABLESPACE <tablespace_name> ENCRYPTION ONLINE USING 'AES256' ENCRYPT;
  ```

SHOULD:
- For column-level encryption:
  ```sql
  ALTER TABLE <owner>.<table> MODIFY (<column> ENCRYPT USING 'AES256');
  ```
- Rotate TDE master key periodically (managed via option group settings)
- Monitor wallet status regularly

MAY:
- Migrate from KMS to CloudHSM for regulatory requirements
- Implement encryption for all application tablespaces

## Common Issues

- symptoms: "Cannot enable TDE — option not available"
  diagnosis: "TDE requires Enterprise Edition. SE2 does not support TDE."
  resolution: "Migrate to Enterprise Edition (BYOL) to use TDE."

- symptoms: "ORA-28365: wallet is not open"
  diagnosis: "TDE wallet not opened after instance restart or option group change."
  resolution: "RDS should auto-open the wallet. If not, check option group configuration and reboot."

- symptoms: "TDE performance impact"
  diagnosis: "TDE adds CPU overhead for encryption/decryption operations."
  resolution: "Use tablespace-level encryption (less overhead than column-level). Monitor CPU after enabling."

## Safety Ratings
- GREEN: describe-option-groups, describe-db-instances, SELECT from v$encryption_wallet/dba_tablespaces/dba_encrypted_columns, describe-key — read-only inspection
- YELLOW: add-option-to-option-group (TDE), ALTER TABLESPACE ENCRYPTION — recoverable but adds CPU overhead
- RED: delete-db-instance, disabling TDE (may cause data access issues) — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "TDE wallet not open — encrypted data inaccessible"
- "KMS key disabled or pending deletion"
- "Fix involves modifying encryption settings"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: KMS key IDs, CloudHSM cluster details, TDE wallet status (core encryption infrastructure)
- HIGH: database credentials, connection strings, encryption algorithm configuration
- MEDIUM: encrypted tablespace/column inventory, option group configuration

## Prohibited Actions
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest deleting a database instance to fix TDE issues — use snapshots"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"
- "NEVER suggest disabling or deleting KMS keys without understanding impact on all dependent resources"

## Phase 3 — Rollback
- "If TDE wallet fails to open after option group change, reboot the instance"
- "If KMS key was disabled, re-enable immediately"
- "Restore from snapshot if TDE configuration change causes data access issues"

## Output Format

```yaml
root_cause: "tde — <specific_cause>"
evidence:
  - type: option_group
    content: "<TDE option configuration>"
  - type: wallet_status
    content: "<v$encryption_wallet output>"
  - type: encrypted_objects
    content: "<encrypted tablespaces/columns>"
severity: HIGH
mitigation:
  immediate: "Fix TDE configuration or wallet state"
  long_term: "Implement TDE monitoring and key rotation schedule"
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
