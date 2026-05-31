---
title: "D2 — Reader Failover"
description: "Diagnose Aurora reader failover and routing issues"
status: active
severity: MEDIUM
triggers:
  - "reader failover"
  - "reader unavailable"
  - "reader endpoint"
  - "reader not responding"
  - "read replica down"
owner: devops-agent
objective: "Identify and resolve Aurora reader failover and routing issues"
context: "Aurora reader instances can become unavailable due to instance failure, scaling, or maintenance. The reader endpoint uses DNS round-robin and does not automatically remove unhealthy readers from rotation immediately. Applications may connect to an unavailable reader."
---

## Phase 1 — Triage

MUST:
- Check all reader instance status:
  ```
  aws rds describe-db-instances --filters Name=db-cluster-id,Values=<cluster-id> \
    --query 'DBInstances[].{Id:DBInstanceIdentifier,Status:DBInstanceStatus,AZ:AvailabilityZone}'
  ```
- Check reader events:
  ```
  aws rds describe-events --source-identifier <reader-instance-id> --source-type db-instance --duration 1440
  ```
- Verify reader endpoint DNS resolution: `nslookup <reader-endpoint>`
- Check reader replica lag:
  ```
  aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name AuroraReplicaLag \
    --dimensions Name=DBInstanceIdentifier,Value=<reader-instance-id> \
    --start-time <start> --end-time <end> --period 60 --statistics Maximum
  ```

SHOULD:
- Check if reader was promoted to writer (causing reader count to decrease):
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> \
    --query 'DBClusters[0].DBClusterMembers'
  ```
- Check custom endpoints if used:
  ```
  aws rds describe-db-cluster-endpoints --db-cluster-identifier <cluster-id>
  ```
- Verify reader is accepting connections:
  - Aurora MySQL: `SELECT @@innodb_read_only;` (should be 1 on reader)
  - Aurora PostgreSQL: `SELECT pg_is_in_recovery();` (should be true on reader)

MAY:
- Check if Auto Scaling is configured for readers
- Review application connection pool health check configuration

## Phase 2 — Remediate

MUST:
- If reader is down: check events for cause (maintenance, failure, scaling)
- If reader was promoted to writer: add a new reader instance
- Verify remaining readers are healthy and accepting connections

SHOULD:
- Use RDS Proxy for automatic reader health checking and routing
- Configure Aurora Auto Scaling to maintain minimum reader count
- Use individual reader instance endpoints with application-level health checks

MAY:
- Create custom endpoints to exclude specific readers from certain workloads
- Implement circuit breaker pattern in application for reader connections

## Common Issues

- symptoms: "Application errors connecting to reader endpoint"
  diagnosis: "Reader endpoint DNS resolving to an unavailable reader instance."
  resolution: "Use RDS Proxy. Implement connection retry. Use instance endpoints with health checks."

- symptoms: "Only one reader and it was promoted to writer"
  diagnosis: "Failover promoted the only reader, leaving no readers for read traffic."
  resolution: "Add reader instances. Configure Auto Scaling with minimum 1 reader."

- symptoms: "Reader endpoint always resolves to same instance"
  diagnosis: "DNS caching. Round-robin only works with fresh DNS lookups."
  resolution: "Reduce DNS TTL. Use RDS Proxy for connection-level distribution."

## Safety Ratings
- GREEN: describe-db-instances, describe-db-clusters, describe-events, describe-db-cluster-endpoints, CloudWatch AuroraReplicaLag/CPUUtilization metrics — read-only inspection
- YELLOW: modify-db-instance, modify-db-cluster-endpoint, add-reader — recoverable configuration changes
- RED: delete-db-instance, force-failover — destructive or high-impact operations in production

## Escalation Conditions
- "Database serves production traffic"
- "Reader unavailability affecting read workloads"
- "Fix requires failover of Aurora cluster"
- "All readers are down leaving no read capacity"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, endpoint addresses
- HIGH: instance status and events (reveal infrastructure topology)
- MEDIUM: CloudWatch metrics, replica lag, Auto Scaling configuration

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix reader issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest force-failover in production without confirming application readiness"
- "NEVER suggest removing the last reader without confirming failover implications"

## Phase 3 — Rollback
- "Restore from snapshot if reader configuration change causes issues"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"
- "If Auto Scaling adds too many readers, adjust scaling policy rather than deleting instances"

## Output Format

```yaml
root_cause: "reader_failover — <specific_cause>"
evidence:
  - type: instance_status
    content: "<reader instance status>"
  - type: rds_events
    content: "<reader events>"
  - type: replica_lag
    content: "<AuroraReplicaLag metric>"
severity: MEDIUM
mitigation:
  immediate: "Restore reader availability"
  long_term: "Configure Auto Scaling and RDS Proxy for reader management"
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
