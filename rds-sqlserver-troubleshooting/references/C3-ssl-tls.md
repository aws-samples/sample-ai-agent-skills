---
title: "C3 — SSL/TLS"
description: "Diagnose SSL/TLS connection issues on RDS SQL Server"
status: active
severity: MEDIUM
triggers:
  - "SSL"
  - "TLS"
  - "encrypted connection"
  - "certificate"
  - "rds-ca"
  - "force encryption"
  - "connection encryption"
owner: devops-agent
objective: "Resolve SSL/TLS configuration and connection issues on RDS SQL Server"
context: "RDS SQL Server supports SSL/TLS encryption for connections in transit. RDS provides a server certificate signed by the RDS CA. Force encryption can be enabled via the rds.force_ssl parameter. Clients must trust the RDS CA certificate. TLS 1.2 is recommended."
---

## Phase 1 — Triage

MUST:
- Check if force SSL is enabled: `aws rds describe-db-parameters --db-parameter-group-name <param-group> --query "Parameters[?ParameterName=='rds.force_ssl']"`
- Check connection encryption status:
  ```sql
  SELECT session_id, encrypt_option, auth_scheme, client_net_address
  FROM sys.dm_exec_connections
  WHERE session_id = @@SPID;
  ```
- Check all connection encryption:
  ```sql
  SELECT encrypt_option, COUNT(*) AS connection_count
  FROM sys.dm_exec_connections
  GROUP BY encrypt_option;
  ```
- Check RDS CA certificate: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].CACertificateIdentifier'`

SHOULD:
- Verify client has the RDS CA certificate installed
- Check TLS version in use:
  ```sql
  SELECT session_id, encrypt_option, protocol_type, protocol_version
  FROM sys.dm_exec_connections WHERE session_id = @@SPID;
  ```
- Check for SSL-related errors in error log:
  ```
  aws rds download-db-log-file-portion --db-instance-identifier <instance-id> --log-file-name error/ERRORLOG
  ```

MAY:
- Verify certificate expiration dates
- Test SSL connection from different client types (SSMS, sqlcmd, JDBC, ODBC)

## Phase 2 — Remediate

MUST:
- Enable force SSL via parameter group:
  ```
  aws rds modify-db-parameter-group \
    --db-parameter-group-name <param-group> \
    --parameters "ParameterName=rds.force_ssl,ParameterValue=1,ApplyMethod=pending-reboot"
  ```
- Download and install RDS CA certificate on clients:
  - Global bundle: `https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem`
  - Region-specific: `https://truststore.pki.rds.amazonaws.com/<region>/<region>-bundle.pem`

SHOULD:
- For SSMS: check "Encrypt connection" and "Trust server certificate" options
- For sqlcmd: use `-N` flag for encrypted connection
  ```
  sqlcmd -S <endpoint>,1433 -U <user> -P <password> -N -C
  ```
- For JDBC connection string:
  ```
  jdbc:sqlserver://<endpoint>:1433;encrypt=true;trustServerCertificate=false;hostNameInCertificate=*.rds.<region>.amazonaws.com
  ```
- For ODBC: set `Encrypt=yes` in connection string
- Rotate CA certificate when RDS updates the CA:
  ```
  aws rds modify-db-instance \
    --db-instance-identifier <instance-id> \
    --ca-certificate-identifier rds-ca-rsa2048-g1 \
    --apply-immediately
  ```

MAY:
- Implement certificate pinning in applications
- Set up monitoring for certificate expiration

## Common Issues

- symptoms: "SSL connection fails — certificate not trusted"
  diagnosis: "Client does not have the RDS CA certificate in its trust store."
  resolution: "Download and install the RDS CA bundle on the client."

- symptoms: "Connections not encrypted despite force_ssl=1"
  diagnosis: "Parameter change requires reboot to take effect."
  resolution: "Reboot the instance after setting rds.force_ssl=1."

- symptoms: "Certificate expired or CA rotation needed"
  diagnosis: "RDS CA certificate approaching expiration."
  resolution: "Update CA certificate identifier on the instance and update client trust stores."

## Safety Ratings
- GREEN: describe-db-parameters (rds.force_ssl), SELECT from sys.dm_exec_connections, describe-db-instances (CACertificateIdentifier) — read-only inspection
- YELLOW: modify-db-parameter-group (rds.force_ssl), modify-db-instance (CA certificate rotation) — recoverable but may require reboot
- RED: delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "SSL enforcement may disconnect clients not configured for SSL"
- "Fix requires parameter group change (rds.force_ssl) that needs reboot"
- "CA certificate rotation affecting all client connections"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: SSL certificates, connection strings with encryption parameters, CA certificate identifiers
- HIGH: database credentials, encryption status of connections
- MEDIUM: TLS version details, cipher suite information

## Prohibited Actions
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest deleting a database instance to fix SSL issues — use snapshots"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest removing SSL enforcement without confirming all clients support the change"

## Phase 3 — Rollback
- "Revert rds.force_ssl parameter and reboot if SSL enforcement breaks clients"
- "Revert CA certificate change if it breaks client trust"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "ssl_tls — <specific_cause>"
evidence:
  - type: ssl_config
    content: "<rds.force_ssl parameter and CA certificate>"
  - type: connection_encryption
    content: "<sys.dm_exec_connections encrypt_option>"
severity: MEDIUM
mitigation:
  immediate: "Fix SSL configuration or client certificate trust"
  long_term: "Enforce SSL for all connections and implement CA rotation process"
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
