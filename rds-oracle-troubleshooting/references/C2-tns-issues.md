---
title: "C2 — TNS Issues"
description: "Diagnose Oracle TNS (Transparent Network Substrate) resolution and connection errors"
status: active
severity: MEDIUM
triggers:
  - "TNS"
  - "ORA-12154"
  - "ORA-12514"
  - "ORA-12505"
  - "ORA-12545"
  - "tnsnames"
  - "service name"
  - "SID"
owner: devops-agent
objective: "Resolve TNS name resolution and service name issues connecting to RDS Oracle"
context: "RDS Oracle uses service names (not SIDs) for connections. The default service name matches the DB name specified at creation. TNS errors on the client side often relate to incorrect connection strings, not RDS configuration. tnsnames.ora is a client-side file — RDS does not use it."
---

## Phase 1 — Triage

MUST:
- Verify the RDS endpoint and port: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].{Endpoint:Endpoint.Address,Port:Endpoint.Port,DBName:DBName}'`
- Verify the connection string format:
  ```
  # Correct format for RDS Oracle
  sqlplus user/password@//<endpoint>:<port>/<service_name>
  # Or using TNS-style
  sqlplus user/password@(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=<endpoint>)(PORT=<port>))(CONNECT_DATA=(SERVICE_NAME=<db_name>)))
  ```
- Check available services on the instance (via SQL if connected):
  ```sql
  SELECT name, network_name FROM v$services;
  ```

SHOULD:
- Verify client tnsnames.ora has correct entry (client-side file)
- Check for DNS resolution of the RDS endpoint
- Verify the DB name (service name): `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].DBName'`
- Check if using SID instead of SERVICE_NAME (common mistake):
  ```
  # WRONG: SID = ORCL
  (CONNECT_DATA=(SID=ORCL))
  # CORRECT: SERVICE_NAME = ORCL
  (CONNECT_DATA=(SERVICE_NAME=ORCL))
  ```

MAY:
- Test with tnsping from client: `tnsping <tns_alias>`
- Check Oracle client version compatibility
- Verify JDBC connection string format for Java applications

## Phase 2 — Remediate

MUST:
- Fix connection string to use SERVICE_NAME (not SID)
- Use the correct RDS endpoint (not IP address — IPs can change on failover)
- Ensure the DB name in the connection string matches the RDS DB name

SHOULD:
- Update client tnsnames.ora with correct RDS entry
- Use Easy Connect syntax for simplicity: `user/password@<endpoint>:<port>/<service_name>`
- For JDBC: `jdbc:oracle:thin:@//<endpoint>:<port>/<service_name>`

MAY:
- Configure Oracle Net connection timeout on client side
- Implement connection string management via configuration files

## Common Issues

- symptoms: "ORA-12154: TNS:could not resolve the connect identifier specified"
  diagnosis: "Client cannot find the TNS alias. tnsnames.ora missing or incorrect."
  resolution: "Fix tnsnames.ora or use Easy Connect syntax with the RDS endpoint directly."

- symptoms: "ORA-12514: TNS:listener does not currently know of service requested"
  diagnosis: "Service name in connection string doesn't match the DB name on RDS."
  resolution: "Use the DB name from describe-db-instances as the SERVICE_NAME."

- symptoms: "ORA-12505: TNS:listener does not currently know of SID given"
  diagnosis: "Using SID instead of SERVICE_NAME in the connection string."
  resolution: "Change CONNECT_DATA to use SERVICE_NAME instead of SID."

- symptoms: "ORA-12545: Connect failed because target host or object does not exist"
  diagnosis: "DNS resolution failure or incorrect hostname."
  resolution: "Verify the RDS endpoint hostname. Check DNS resolution from the client."

## Safety Ratings
- GREEN: describe-db-instances (endpoint, port, DBName), SELECT from v$services, tnsping — read-only inspection
- YELLOW: application connection string changes — recoverable configuration changes
- RED: delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "TNS issues affecting multiple application connections"
- "Fix requires application configuration changes across multiple services"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings (contain passwords), TNS configuration
- HIGH: RDS endpoint addresses (reveal infrastructure topology)
- MEDIUM: service names, port configuration

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix TNS issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest using IP addresses instead of RDS endpoints (IPs change on failover)"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"

## Phase 3 — Rollback
- "Revert application connection string changes if new configuration causes issues"
- "Revert tnsnames.ora changes on client side if they break connectivity"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "tns_issue — <specific_cause>"
evidence:
  - type: connection_string
    content: "<current connection string>"
  - type: rds_config
    content: "<endpoint, port, DB name>"
severity: MEDIUM
mitigation:
  immediate: "Fix connection string with correct endpoint and service name"
  long_term: "Standardize connection string management"
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
