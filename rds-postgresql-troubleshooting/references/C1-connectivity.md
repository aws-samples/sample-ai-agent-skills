---
title: "C1 — RDS PostgreSQL Connectivity Issues"
description: "Diagnose connection failures, timeouts, and authentication errors"
status: active
severity: HIGH
triggers:
  - "cannot connect"
  - "connection refused"
  - "connection timed out"
  - "FATAL: password authentication failed"
---

## Phase 1 — Triage

MUST:
- Check instance status: `aws rds describe-db-instances --db-instance-identifier <id> --query 'DBInstances[0].DBInstanceStatus'`
- Verify security group allows port 5432: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Check DatabaseConnections metric for max_connections exhaustion
- Test DNS resolution: `nslookup <endpoint>`
- Check error log for auth failures: `aws rds download-db-log-file-portion --db-instance-identifier <id> --log-file-name error/postgresql.log`

SHOULD:
- Check `PubliclyAccessible` setting
- Verify NACLs on subnet
- Check for SSL requirement: `rds.force_ssl=1` in parameter group
- Test connection: `psql -h <endpoint> -U <user> -d <database> -p 5432`

## Phase 2 — Remediate

- Security group: add inbound rule for port 5432
- Auth failure: reset password `aws rds modify-db-instance --master-user-password <new-pass>`
- SSL required: use `sslmode=require` in connection string
- Connection timeout: check route tables, NAT gateway, VPC peering
- max_connections: use RDS Proxy or scale instance

## Safety Ratings
- GREEN: describe-db-instances, CloudWatch DatabaseConnections metrics, SELECT from pg_stat_activity, nslookup — read-only inspection
- YELLOW: modify-db-instance (security group), pg_terminate_backend() — recoverable but impacts active sessions
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Connection failures causing application outages"
- "Fix requires parameter group change that needs reboot"
- "Fix requires killing active database sessions"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, pg_stat_activity details (contain user/query info)
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
