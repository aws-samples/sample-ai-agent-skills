---
title: "C1 — Connection Failures"
description: "Diagnose connection failures to RDS Oracle instances"
status: active
severity: HIGH
triggers:
  - "cannot connect"
  - "connection refused"
  - "connection timeout"
  - "ORA-12170"
  - "ORA-12541"
  - "ORA-01017"
  - "DatabaseConnections"
owner: devops-agent
objective: "Identify and resolve connectivity issues to RDS Oracle instances"
context: "Connection failures can stem from security group rules, network ACLs, VPC routing, DNS resolution, Oracle listener state, max connection limits, or authentication issues. RDS Oracle listens on port 1521 by default."
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
- Check session/process limits via SQL:
  ```sql
  SELECT resource_name, current_utilization, max_utilization, limit_value
  FROM v$resource_limit WHERE resource_name IN ('sessions','processes');
  ```

MAY:
- Test connectivity from an EC2 instance in the same VPC
- Check for VPC peering or Transit Gateway routing issues
- Verify SSL/TLS configuration if encrypted connections required

## Phase 2 — Remediate

MUST:
- For security group: add inbound rule for port 1521 (or custom port) from client CIDR
- For connection limit: increase `processes` parameter (requires reboot) or kill idle sessions
  ```sql
  -- Find idle sessions
  SELECT sid, serial#, username, status, last_call_et/60 AS idle_minutes
  FROM v$session WHERE status = 'INACTIVE' AND username IS NOT NULL
  ORDER BY last_call_et DESC;
  -- Kill idle session
  EXEC rdsadmin.rdsadmin_util.kill('<sid>', '<serial#>');
  ```
- For authentication failures (ORA-01017): verify username/password, check case sensitivity

SHOULD:
- Implement connection pooling to manage connection count
- Set up CloudWatch alarm on DatabaseConnections approaching limit
- For public access: ensure internet gateway and public subnet routing

MAY:
- Configure connection timeout parameters in the application
- Use RDS Proxy for connection management (if supported for Oracle)

## Common Issues

- symptoms: "ORA-12170: TNS:Connect timeout occurred"
  diagnosis: "Network-level block. Security group, NACL, or routing preventing connection."
  resolution: "Check security group inbound rules for port 1521. Verify subnet routing."

- symptoms: "ORA-01017: invalid username/password; logon denied"
  diagnosis: "Wrong credentials or case-sensitive password."
  resolution: "Verify credentials. Oracle 12c+ has case-sensitive passwords by default."

- symptoms: "ORA-00018: maximum number of sessions exceeded"
  diagnosis: "Connection limit reached. Default sessions = 1.5 * processes + 22."
  resolution: "Increase processes parameter. Kill idle sessions. Implement connection pooling."

## Safety Ratings
- GREEN: describe-db-instances, describe-events, CloudWatch DatabaseConnections metrics, SELECT from v$resource_limit/v$session — read-only inspection
- YELLOW: modify-db-instance (security group), EXEC rdsadmin.rdsadmin_util.kill — recoverable but impacts active sessions
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Connection limit reached causing application outages"
- "Fix requires parameter group change (processes) that needs reboot"
- "Fix requires killing active database sessions"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, v$session details (contain username, program, machine)
- HIGH: security group rules (expose network topology)
- MEDIUM: connection count metrics, resource limit utilization

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix connection issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest opening security groups to 0.0.0.0/0 to fix connectivity"
- "NEVER suggest ALTER SYSTEM commands that could crash the instance"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
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
