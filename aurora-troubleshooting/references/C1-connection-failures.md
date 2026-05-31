---
title: "C1 — Connection Failures"
description: "Diagnose connection failures to Aurora clusters"
status: active
severity: HIGH
triggers:
  - "connection refused"
  - "connection timeout"
  - "cannot connect"
  - "access denied"
  - "authentication failed"
  - "too many connections"
owner: devops-agent
objective: "Identify and resolve connection failures to Aurora clusters"
context: "Aurora connection failures can stem from security group rules, VPC configuration, endpoint misconfiguration, authentication issues, connection limits, or instance unavailability. Aurora has multiple endpoint types that must be used correctly."
---

## Phase 1 — Triage

MUST:
- Check cluster and instance status:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].Status'
  aws rds describe-db-instances --filters Name=db-cluster-id,Values=<cluster-id> \
    --query 'DBInstances[].{Id:DBInstanceIdentifier,Status:DBInstanceStatus}'
  ```
- Check security group rules:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].VpcSecurityGroups'
  aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[0].IpPermissions'
  ```
- Check connection count:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections \
    --dimensions Name=DBInstanceIdentifier,Value=<instance-id> \
    --start-time <start> --end-time <end> --period 300 --statistics Maximum
  ```
- Verify endpoint DNS resolution: `nslookup <cluster-endpoint>`

SHOULD:
- Check VPC subnet routing and NACLs
- Verify the client is in the same VPC or has proper network path (VPN, peering, Transit Gateway)
- For Aurora MySQL — check authentication:
  ```sql
  SELECT user, host, plugin FROM mysql.user WHERE user = '<username>';
  ```
- For Aurora PostgreSQL — check authentication:
  ```sql
  SELECT usename, valuntil FROM pg_user WHERE usename = '<username>';
  SELECT * FROM pg_hba_file_rules;
  ```

MAY:
- Test connectivity with telnet or nc: `nc -zv <endpoint> <port>`
- Check for SSL/TLS requirements: `aws rds describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].{IAMAuth:IAMDatabaseAuthenticationEnabled}'`

## Phase 2 — Remediate

MUST:
- For security group issues: add inbound rule for the client IP/CIDR on port 3306 (MySQL) or 5432 (PostgreSQL)
- For authentication failures: verify username/password, check user host restrictions
- For connection limit exceeded: increase `max_connections` or implement connection pooling

SHOULD:
- Use RDS Proxy for connection pooling and management
- Enable IAM database authentication for token-based access
- Configure SSL/TLS for encrypted connections

MAY:
- Implement connection retry logic in the application
- Set up CloudWatch alarms for DatabaseConnections approaching limits

## Common Issues

- symptoms: "Connection timed out"
  diagnosis: "Security group, NACL, or routing issue. Client cannot reach Aurora endpoint."
  resolution: "Verify security group allows inbound on correct port. Check VPC routing."

- symptoms: "Access denied for user"
  diagnosis: "Wrong password, user doesn't exist, or host restriction."
  resolution: "Verify credentials. Check user host restrictions (MySQL) or pg_hba.conf (PostgreSQL)."

- symptoms: "Too many connections"
  diagnosis: "Connection count exceeds max_connections for the instance class."
  resolution: "Implement connection pooling (RDS Proxy). Scale up instance. Increase max_connections."

## Safety Ratings
- GREEN: describe-db-clusters, describe-db-instances, CloudWatch DatabaseConnections metrics, nslookup, SHOW VARIABLES, pg_stat_activity queries — read-only inspection
- YELLOW: modify-db-instance, modify-db-cluster, modify security group rules — recoverable configuration changes
- RED: delete-db-cluster, force-failover — destructive or high-impact operations in production

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires parameter group change that needs reboot"
- "Fix requires failover of Aurora cluster"
- "Fix involves modifying security group rules affecting other services"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, authentication plugin details (mysql.user, pg_hba_file_rules)
- HIGH: security group rules and VPC configuration (expose network topology)
- MEDIUM: connection count metrics, endpoint DNS resolution results

## Prohibited Actions
- "NEVER suggest deleting a database cluster to fix connection issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest force-failover in production without confirming application readiness"
- "NEVER suggest opening security groups to 0.0.0.0/0 to fix connectivity"

## Phase 3 — Rollback
- "Restore from snapshot if parameter change causes issues"
- "Revert security group changes if they break other services"
- "Revert parameter group changes and reboot if needed"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "connection_failure — <specific_cause>"
evidence:
  - type: cluster_status
    content: "<cluster and instance status>"
  - type: security_group
    content: "<security group rules>"
  - type: connection_count
    content: "<DatabaseConnections metric>"
severity: HIGH
mitigation:
  immediate: "Fix connectivity or authentication issue"
  long_term: "Implement RDS Proxy and connection monitoring"
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
