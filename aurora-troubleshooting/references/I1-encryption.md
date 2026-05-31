---
title: "I1 — Encryption"
description: "Diagnose Aurora encryption issues (at-rest and in-transit)"
status: active
severity: HIGH
triggers:
  - "encryption"
  - "KMS"
  - "encrypted"
  - "SSL"
  - "TLS"
  - "at-rest encryption"
  - "in-transit encryption"
owner: devops-agent
objective: "Identify and resolve Aurora encryption configuration issues"
context: "Aurora supports encryption at rest (KMS) and in transit (SSL/TLS). At-rest encryption must be enabled at cluster creation and cannot be changed later. All instances and snapshots inherit the cluster encryption setting. In-transit encryption uses SSL/TLS certificates. Aurora supports both AWS-managed and customer-managed KMS keys."
---

## Phase 1 — Triage

MUST:
- Check cluster encryption status:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> \
    --query 'DBClusters[0].{Encrypted:StorageEncrypted,KmsKeyId:KmsKeyId}'
  ```
- Check KMS key status:
  ```
  aws kms describe-key --key-id <kms-key-id> --query 'KeyMetadata.{State:KeyState,Enabled:Enabled}'
  ```
- Check SSL/TLS configuration:
  - Aurora MySQL:
    ```sql
    SHOW VARIABLES LIKE 'require_secure_transport';
    SHOW STATUS LIKE 'Ssl_cipher';
    ```
  - Aurora PostgreSQL:
    ```sql
    SHOW ssl;
    SELECT * FROM pg_stat_ssl WHERE pid = pg_backend_pid();
    ```
- Check cluster parameter for SSL enforcement:
  ```
  aws rds describe-db-cluster-parameters --db-cluster-parameter-group-name <cluster-param-group> \
    --query 'Parameters[?ParameterName==`require_secure_transport` || ParameterName==`rds.force_ssl`]'
  ```

SHOULD:
- Verify KMS key policy allows RDS service access
- Check if SSL certificate is valid and not expired
- Verify client is using the correct RDS CA certificate bundle

MAY:
- Check CloudTrail for KMS key usage events
- Review cross-account KMS key sharing if applicable

## Phase 2 — Remediate

MUST:
- For KMS key disabled: re-enable the KMS key: `aws kms enable-key --key-id <key-id>`
- For KMS key pending deletion: cancel deletion: `aws kms cancel-key-deletion --key-id <key-id>`
- For unencrypted cluster needing encryption: create an encrypted snapshot copy and restore to a new cluster

SHOULD:
- Enforce SSL/TLS:
  - Aurora MySQL: set `require_secure_transport = 1` in cluster parameter group
  - Aurora PostgreSQL: set `rds.force_ssl = 1` in cluster parameter group
- Download the latest RDS CA certificate bundle for client connections
- Rotate KMS keys periodically (automatic rotation available for customer-managed keys)

MAY:
- Implement IAM database authentication for token-based SSL connections
- Use customer-managed KMS keys for cross-account access control

## Common Issues

- symptoms: "Cannot create encrypted snapshot copy"
  diagnosis: "KMS key not available in target region or insufficient permissions."
  resolution: "Create a KMS key in the target region. Grant appropriate permissions."

- symptoms: "SSL connection required but client not using SSL"
  diagnosis: "require_secure_transport (MySQL) or rds.force_ssl (PostgreSQL) is enabled."
  resolution: "Configure client to use SSL with the RDS CA certificate bundle."

- symptoms: "KMS key is pending deletion — cluster inaccessible"
  diagnosis: "KMS key scheduled for deletion. Cluster cannot decrypt data."
  resolution: "Cancel key deletion immediately: aws kms cancel-key-deletion."

## Safety Ratings
- GREEN: describe-db-clusters (encryption status), describe-key, SHOW VARIABLES/ssl, pg_stat_ssl — read-only inspection
- YELLOW: modify-db-cluster (enable IAM auth), modify-db-cluster-parameter-group (force SSL) — recoverable changes
- RED: kms disable-key, kms schedule-key-deletion — extremely destructive, can make cluster permanently inaccessible

## Escalation Conditions
- "Database serves production traffic"
- "KMS key is disabled or pending deletion — cluster may become inaccessible"
- "Fix requires parameter group change to enforce SSL (needs reboot)"
- "Fix involves modifying encryption settings"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: KMS key IDs, key policies, encryption configuration (core security infrastructure)
- HIGH: SSL/TLS certificates, connection strings with SSL parameters
- HIGH: database credentials, IAM authentication configuration

## Prohibited Actions
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest deleting or disabling KMS keys without understanding the impact on all dependent resources"
- "NEVER suggest removing SSL enforcement without confirming all clients support unencrypted connections"
- "NEVER suggest storing KMS key material in application code or configuration files"

## Phase 3 — Rollback
- "If KMS key was disabled, re-enable immediately: aws kms enable-key"
- "If KMS key deletion was scheduled, cancel immediately: aws kms cancel-key-deletion"
- "Revert SSL enforcement parameter changes and reboot if needed"
- "Restore from snapshot if encryption configuration change causes issues"

## Output Format

```yaml
root_cause: "encryption_issue — <specific_cause>"
evidence:
  - type: encryption_config
    content: "<cluster encryption status>"
  - type: kms_key
    content: "<KMS key status>"
  - type: ssl_config
    content: "<SSL/TLS configuration>"
severity: HIGH
mitigation:
  immediate: "Resolve KMS key or SSL configuration issue"
  long_term: "Implement encryption monitoring and key rotation"
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
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling encryption"
  - "NEVER force failover without understanding impact"
