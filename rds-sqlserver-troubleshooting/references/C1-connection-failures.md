---
title: "C1 — Connection Failures"
description: "Diagnose connection failures to RDS SQL Server instances"
status: active
severity: HIGH
triggers:
  - "cannot connect"
  - "connection refused"
  - "connection timeout"
  - "login failed"
  - "DatabaseConnections"
  - "Error 18456"
  - "Error 4060"
owner: devops-agent
objective: "Identify and resolve connectivity issues to RDS SQL Server instances"
context: "Connection failures can stem from security group rules, network ACLs, VPC routing, DNS resolution, SQL Server state, max connection limits, or authentication issues. RDS SQL Server listens on port 1433 by default."
---

## Phase 1 — Triage

MUST:
- Check instance status: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].{Status:DBInstanceStatus,Endpoint:Endpoint,Port:Endpoint.Port}'`
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[0].IpPermissions'`
- Check current connections: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check RDS events for connectivity issues: `aws rds describe-events --source-identifier <instance-id> --source-type db-instance --duration 1440`
- Verify DNS resolution of endpoint: endpoint format is `<instance-id>.<random>.rds.<region>.amazonaws.com`

SHOULD:
- Check VPC subnet routing: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`
- Check network ACLs: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`
- Check if instance is publicly accessible: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].PubliclyAccessible'`
- Check connection count and limits via T-SQL:
  ```sql
  SELECT COUNT(*) AS current_connections FROM sys.dm_exec_connections;
  SELECT name, value_in_use FROM sys.configurations WHERE name = 'user connections';
  -- 0 means SQL Server manages dynamically (default, max ~32,767)
  ```

MAY:
- Test connectivity from an EC2 instance in the same VPC
- Check for VPC peering or Transit Gateway routing issues
- Verify SSL/TLS configuration if encrypted connections required

## Phase 2 — Remediate

MUST:
- For security group: add inbound rule for port 1433 (or custom port) from client CIDR
- For connection limit: kill idle sessions or scale instance class
  ```sql
  -- Find idle sessions
  SELECT s.session_id, s.login_name, s.host_name, s.program_name,
         s.status, s.last_request_start_time
  FROM sys.dm_exec_sessions s
  WHERE s.is_user_process = 1 AND s.status = 'sleeping'
  ORDER BY s.last_request_start_time ASC;
  -- Kill idle session
  KILL <session_id>;
  ```
- For authentication failures (Error 18456): verify username/password, check login exists

SHOULD:
- Implement connection pooling to manage connection count
- Set up CloudWatch alarm on DatabaseConnections approaching limit
- For public access: ensure internet gateway and public subnet routing

MAY:
- Configure connection timeout in the application connection string
- Use RDS Proxy for connection management

## Common Issues

- symptoms: "Connection timeout — cannot reach server"
  diagnosis: "Network-level block. Security group, NACL, or routing preventing connection."
  resolution: "Check security group inbound rules for port 1433. Verify subnet routing."

- symptoms: "Error 18456: Login failed for user"
  diagnosis: "Wrong credentials, login disabled, or login does not exist."
  resolution: "Verify credentials. Check login exists: SELECT name, is_disabled FROM sys.server_principals WHERE type = 'S';"

- symptoms: "Error 4060: Cannot open database requested by the login"
  diagnosis: "Database does not exist or login does not have access."
  resolution: "Verify database name. Grant access: USE [master]; ALTER LOGIN [user] WITH DEFAULT_DATABASE = [dbname];"

## Safety Ratings
- GREEN: describe-db-instances, CloudWatch DatabaseConnections metrics, SELECT from sys.dm_exec_connections/sys.dm_exec_sessions/sys.server_principals — read-only inspection
- YELLOW: modify-db-instance (security group), KILL session — recoverable but impacts active sessions
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Connection failures causing application outages"
- "Fix requires parameter group change that needs reboot"
- "Fix requires killing active database sessions"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, login details from sys.server_principals
- HIGH: security group rules (expose network topology)
- MEDIUM: connection count metrics, session details

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix connection issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest opening security groups to 0.0.0.0/0 to fix connectivity"
- "NEVER suggest granting sysadmin role"

## Phase 3 — Rollback
- "Revert security group changes if they break other services"
- "Revert parameter group changes and reboot if needed"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "connection_failure — <specific_cause>"
evidence:
  - type: instance_status
    content: "<instance status and endpoint>"
  - type: security_group
    content: "<inbound rules>"
  - type: connections
    content: "<DatabaseConnections metric>"
severity: HIGH
mitigation:
  immediate: "Fix security group or kill idle sessions"
  long_term: "Implement connection pooling and monitoring"
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
