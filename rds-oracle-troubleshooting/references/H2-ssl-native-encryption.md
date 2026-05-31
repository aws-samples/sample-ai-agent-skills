---
title: "H2 — SSL and Native Network Encryption"
description: "Diagnose Oracle SSL/TLS and Native Network Encryption issues on RDS Oracle"
status: active
severity: MEDIUM
triggers:
  - "SSL"
  - "TLS"
  - "native network encryption"
  - "NNE"
  - "sqlnet encryption"
  - "ORA-28860"
  - "ORA-28864"
  - "certificate"
  - "in-transit encryption"
owner: devops-agent
objective: "Resolve SSL/TLS and Oracle Native Network Encryption configuration issues"
context: "RDS Oracle supports two in-transit encryption methods: Oracle Native Network Encryption (NNE, configured via option group, no certificates) and SSL (requires certificates and option group). NNE is simpler. SSL provides certificate-based authentication. Both are configured via option groups, not sqlnet.ora."
---

## Phase 1 — Triage

MUST:
- Check option group for encryption options:
  ```
  aws rds describe-option-groups --option-group-name <option-group> \
    --query "OptionGroupsList[0].Options[?OptionName=='NATIVE_NETWORK_ENCRYPTION' || OptionName=='SSL']"
  ```
- Check current encryption status from Oracle:
  ```sql
  SELECT network_service_banner FROM v$session_connect_info WHERE sid = SYS_CONTEXT('USERENV','SID');
  ```
- Verify option group association: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].OptionGroupMemberships'`

SHOULD:
- For SSL: check certificate validity
  ```sql
  SELECT * FROM v$ssl_ciphers WHERE name IS NOT NULL;
  ```
- Check NNE option settings:
  ```
  aws rds describe-option-groups --option-group-name <option-group> \
    --query "OptionGroupsList[0].Options[?OptionName=='NATIVE_NETWORK_ENCRYPTION'].OptionSettings"
  ```
- Verify client-side configuration (Oracle client must support the encryption algorithm)

MAY:
- Check for encryption-related errors in alert log
- Verify JDBC thin client SSL configuration

## Phase 2 — Remediate

MUST:
- Enable Native Network Encryption via option group:
  ```
  aws rds add-option-to-option-group \
    --option-group-name <option-group> \
    --options '[{
      "OptionName": "NATIVE_NETWORK_ENCRYPTION",
      "OptionSettings": [
        {"Name": "SQLNET.ENCRYPTION_SERVER", "Value": "REQUIRED"},
        {"Name": "SQLNET.ENCRYPTION_TYPES_SERVER", "Value": "AES256"},
        {"Name": "SQLNET.CRYPTO_CHECKSUM_SERVER", "Value": "REQUIRED"},
        {"Name": "SQLNET.CRYPTO_CHECKSUM_TYPES_SERVER", "Value": "SHA256"}
      ]
    }]'
  ```
- For SSL: add SSL option and download RDS CA certificate
  ```
  aws rds add-option-to-option-group \
    --option-group-name <option-group> \
    --options '[{"OptionName": "SSL", "OptionSettings": [{"Name": "SQLNET.SSL_VERSION", "Value": "1.2"}]}]'
  ```

SHOULD:
- For SSL client connection:
  ```
  sqlplus user/password@(DESCRIPTION=(ADDRESS=(PROTOCOL=TCPS)(HOST=<endpoint>)(PORT=2484))(CONNECT_DATA=(SERVICE_NAME=<db_name>))(SECURITY=(SSL_SERVER_CERT_DN="C=US,ST=Washington,L=Seattle,O=Amazon.com,OU=RDS,CN=<endpoint>")))
  ```
- Note: SSL uses port 2484 by default (not 1521)
- Download RDS CA bundle for client trust store

MAY:
- Configure JDBC for SSL:
  ```
  jdbc:oracle:thin:@(DESCRIPTION=(ADDRESS=(PROTOCOL=TCPS)(HOST=<endpoint>)(PORT=2484))(CONNECT_DATA=(SERVICE_NAME=<db_name>)))
  ```
- Implement mutual TLS (mTLS) for additional security

## Common Issues

- symptoms: "ORA-28860: Fatal SSL error"
  diagnosis: "SSL configuration mismatch between client and server."
  resolution: "Verify SSL version compatibility. Check certificate trust chain. Use correct port (2484)."

- symptoms: "Connection works without encryption despite NNE being configured"
  diagnosis: "NNE server set to REQUESTED (not REQUIRED). Client not requesting encryption."
  resolution: "Set SQLNET.ENCRYPTION_SERVER to REQUIRED to enforce encryption."

- symptoms: "Performance degradation after enabling encryption"
  diagnosis: "Encryption adds CPU overhead. AES256 is more CPU-intensive than AES128."
  resolution: "Monitor CPU. Consider AES128 if AES256 overhead is too high. Ensure instance has adequate CPU."

## Safety Ratings
- GREEN: describe-option-groups, SELECT from v$session_connect_info/v$ssl_ciphers — read-only inspection
- YELLOW: add-option-to-option-group (NNE/SSL), modify option settings — recoverable but may affect active connections
- RED: delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Encryption configuration change may disconnect active clients"
- "Fix requires option group changes affecting all connections"
- "SSL certificate issues affecting application connectivity"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: SSL certificates, encryption algorithm configuration, connection strings with SSL parameters
- HIGH: database credentials, option group security settings
- MEDIUM: encryption status, cipher suite details

## Prohibited Actions
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest deleting a database instance to fix encryption issues — use snapshots"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"
- "NEVER suggest removing encryption requirements without confirming all clients support the change"

## Phase 3 — Rollback
- "Revert option group encryption settings if they break client connectivity"
- "If NNE set to REQUIRED breaks clients, temporarily set to REQUESTED"
- "Restore from snapshot if encryption configuration change causes issues"

## Output Format

```yaml
root_cause: "encryption — <specific_cause>"
evidence:
  - type: option_group
    content: "<encryption option configuration>"
  - type: session_encryption
    content: "<v$session_connect_info output>"
severity: MEDIUM
mitigation:
  immediate: "Fix encryption configuration in option group"
  long_term: "Enforce encryption for all connections"
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
