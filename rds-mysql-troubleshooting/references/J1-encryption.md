---
title: "J1 — Encryption Configuration"
description: "Diagnose encryption at rest and in transit issues for RDS MySQL"
status: active
severity: HIGH
triggers:
  - "encryption"
  - "KMS"
  - "SSL"
  - "TLS"
  - "unencrypted"
---

## Phase 1 — Triage

MUST:
- Check encryption status: `aws rds describe-db-instances --db-instance-identifier <id> --query 'DBInstances[0].[StorageEncrypted,KmsKeyId]'`
- Check KMS key status: `aws kms describe-key --key-id <key-id> --query 'KeyMetadata.[KeyState,Enabled]'`
- Check SSL/TLS connection:
  ```sql
  SHOW VARIABLES LIKE '%ssl%';
  SELECT * FROM performance_schema.session_status WHERE VARIABLE_NAME = 'Ssl_cipher';
  ```
- Verify `require_secure_transport` parameter if enforcing TLS

SHOULD:
- Check KMS key policy allows RDS service: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Verify cross-region KMS key for cross-region replicas
- Check certificate bundle for client connections

## Phase 2 — Remediate

- Encrypt unencrypted instance: create snapshot → copy with encryption → restore
  ```
  aws rds create-db-snapshot --db-instance-identifier <id> --db-snapshot-identifier <snap-id>
  aws rds copy-db-snapshot --source-db-snapshot-identifier <snap-id> --target-db-snapshot-identifier <encrypted-snap> --kms-key-id <key-id>
  aws rds restore-db-instance-from-db-snapshot --db-instance-identifier <new-id> --db-snapshot-identifier <encrypted-snap>
  ```
- Enforce TLS: set `require_secure_transport=1` in parameter group
- Download RDS CA bundle for client verification: `https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem`
- KMS key disabled: re-enable key or restore from snapshot with different key

## Safety Ratings
- GREEN: describe-db-instances (StorageEncrypted, KmsKeyId), describe-key, SHOW VARIABLES LIKE 'require_secure_transport' — read-only inspection
- YELLOW: modify-db-parameter-group (require_secure_transport), modify-db-instance (CA certificate) — recoverable changes
- RED: kms disable-key, kms schedule-key-deletion, delete-db-instance — extremely destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "KMS key disabled or pending deletion — instance may become inaccessible"
- "Fix requires parameter group change to enforce SSL (needs reboot)"
- "Fix involves modifying encryption settings"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: KMS key IDs, key policies, encryption configuration (core security infrastructure)
- HIGH: SSL/TLS certificates, connection strings with SSL parameters
- HIGH: database credentials

## Prohibited Actions
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest deleting or disabling KMS keys without understanding impact"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"
- "NEVER suggest removing SSL enforcement without confirming all clients support the change"

## Phase 3 — Rollback
- "If KMS key was disabled, re-enable immediately: aws kms enable-key"
- "If KMS key deletion was scheduled, cancel immediately: aws kms cancel-key-deletion"
- "Revert SSL enforcement parameter changes and reboot if needed"

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
