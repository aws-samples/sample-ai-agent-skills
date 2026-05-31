---
title: "C2 — Endpoint Confusion"
description: "Diagnose issues caused by incorrect Aurora endpoint usage"
status: active
severity: MEDIUM
triggers:
  - "wrong endpoint"
  - "writes to reader"
  - "read-only connection"
  - "endpoint confusion"
  - "which endpoint"
  - "custom endpoint"
owner: devops-agent
objective: "Identify and resolve issues caused by using incorrect Aurora endpoints"
context: "Aurora provides multiple endpoint types: cluster endpoint (writer), reader endpoint (DNS round-robin), instance endpoints, and custom endpoints. Using the wrong endpoint for a workload causes write failures on readers, unbalanced load, or connectivity issues after failover."
---

## Phase 1 — Triage

MUST:
- List all endpoints:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> \
    --query 'DBClusters[0].{WriterEndpoint:Endpoint,ReaderEndpoint:ReaderEndpoint,Port:Port}'
  aws rds describe-db-cluster-endpoints --db-cluster-identifier <cluster-id>
  ```
- Identify current writer:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> \
    --query 'DBClusters[0].DBClusterMembers[?IsClusterWriter==`true`].DBInstanceIdentifier'
  ```
- Check what endpoint the application is using (review application configuration)
- Verify DNS resolution for each endpoint: `nslookup <endpoint>`

SHOULD:
- Check if application is sending writes to a reader endpoint:
  - Aurora MySQL: `SELECT @@innodb_read_only;` (1 = reader, 0 = writer)
  - Aurora PostgreSQL: `SELECT pg_is_in_recovery();` (true = reader, false = writer)
- Check custom endpoint configuration:
  ```
  aws rds describe-db-cluster-endpoints --db-cluster-identifier <cluster-id> \
    --query 'DBClusterEndpoints[?EndpointType!=`WRITER` && EndpointType!=`READER`]'
  ```

MAY:
- Review application connection string patterns
- Check if RDS Proxy is configured for endpoint routing

## Phase 2 — Remediate

MUST:
- Use cluster endpoint for all write operations
- Use reader endpoint or RDS Proxy for read operations
- Never hardcode instance endpoints (they don't follow failover)

SHOULD:
- Implement read/write splitting in the application
- Use RDS Proxy for automatic read/write routing
- Create custom endpoints for workload isolation (e.g., analytics readers)

MAY:
- Implement application-level endpoint selection with health checks
- Use connection string parameters to enforce read-only mode on reader connections

## Common Issues

- symptoms: "ERROR 1290: The MySQL server is running with the --read-only option"
  diagnosis: "Application sending writes to a reader instance."
  resolution: "Use the cluster (writer) endpoint for write operations."

- symptoms: "ERROR: cannot execute INSERT in a read-only transaction (PostgreSQL)"
  diagnosis: "Application connected to a reader instance attempting writes."
  resolution: "Use the cluster (writer) endpoint for write operations."

- symptoms: "After failover, application still connects to old writer (now reader)"
  diagnosis: "Application using instance endpoint instead of cluster endpoint."
  resolution: "Switch to cluster endpoint. Instance endpoints don't follow failover."

## Safety Ratings
- GREEN: describe-db-clusters, describe-db-cluster-endpoints, nslookup, SELECT @@innodb_read_only, pg_is_in_recovery() — read-only inspection
- YELLOW: modify-db-cluster-endpoint, application connection string changes — recoverable configuration changes
- RED: delete-db-cluster, force-failover — destructive or high-impact operations in production

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires application configuration changes affecting active connections"
- "Fix requires failover of Aurora cluster"
- "Writes are being sent to reader endpoints causing data integrity concerns"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, application configuration (may contain passwords)
- HIGH: endpoint addresses and DNS resolution (reveal internal infrastructure)
- MEDIUM: instance roles (writer/reader), custom endpoint membership

## Prohibited Actions
- "NEVER suggest deleting a database cluster to fix endpoint issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest hardcoding instance endpoints — they don't follow failover"
- "NEVER suggest force-failover in production without confirming application readiness"

## Phase 3 — Rollback
- "Restore from snapshot if configuration change causes issues"
- "Revert application connection string changes if new endpoint causes problems"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "endpoint_confusion — <specific_cause>"
evidence:
  - type: endpoints
    content: "<cluster endpoint configuration>"
  - type: application_config
    content: "<application connection string>"
  - type: instance_role
    content: "<writer/reader status>"
severity: MEDIUM
mitigation:
  immediate: "Correct endpoint usage in application"
  long_term: "Implement RDS Proxy or proper read/write splitting"
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
