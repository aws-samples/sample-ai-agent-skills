---
title: "C1 — RDS MySQL Connectivity Issues"
description: "Diagnose connection failures, timeouts, and max_connections exhaustion"
status: active
severity: HIGH
triggers:
  - "cannot connect"
  - "connection refused"
  - "connection timed out"
  - "Too many connections"
  - "max_connections"
---

## Phase 1 — Triage

MUST:
- Check instance status: `aws rds describe-db-instances --db-instance-identifier <id> --query 'DBInstances[0].DBInstanceStatus'`
- Check DatabaseConnections metric: `aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections --dimensions Name=DBInstanceIdentifier,Value=<id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Verify security group allows inbound on port 3306: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[0].IpPermissions'`
- Check subnet group and VPC routing: `aws rds describe-db-instances --db-instance-identifier <id> --query 'DBInstances[0].[DBSubnetGroup,VpcSecurityGroups]'`
- Test DNS resolution: `nslookup <endpoint>`

SHOULD:
- Check if instance is in a private subnet (no public IP): verify `PubliclyAccessible` setting
- Check NACLs on the subnet: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`
- Check current connections:
  ```sql
  SHOW GLOBAL STATUS LIKE 'Threads_connected';
  SHOW VARIABLES LIKE 'max_connections';
  SELECT user, host, COUNT(*) FROM information_schema.PROCESSLIST GROUP BY user, host;
  ```

## Phase 2 — Remediate

- Security group: add inbound rule for port 3306 from client CIDR/SG
- Too many connections: kill idle connections `CALL mysql.rds_kill(<id>)`, use RDS Proxy, or scale instance
- DNS resolution: ensure VPC DNS resolution is enabled
- Public access: set `PubliclyAccessible=true` if needed, or use VPN/bastion
- Connection timeout: check route tables, NAT gateway, VPC peering routes

## Safety Ratings
- GREEN: describe-db-instances, CloudWatch DatabaseConnections metrics, SHOW GLOBAL STATUS, SHOW VARIABLES, nslookup — read-only inspection
- YELLOW: modify-db-instance (security group), CALL mysql.rds_kill() — recoverable but impacts active sessions
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Connection failures causing application outages"
- "Fix requires parameter group change that needs reboot"
- "Fix requires killing active database sessions"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, processlist details (contain user/host/query)
- HIGH: security group rules (expose network topology)
- MEDIUM: connection count metrics, DNS resolution results

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix connection issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest opening security groups to 0.0.0.0/0 to fix connectivity"
- "NEVER suggest setting innodb_force_recovery or modifying pg_hba.conf directly"

## Phase 3 — Rollback
- "Revert security group changes if they break other services"
- "Revert parameter group changes and reboot if needed"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

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
