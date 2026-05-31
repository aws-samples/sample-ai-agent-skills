---
title: "A3 — Aurora Cluster Endpoint Issues"
description: "Diagnose issues with Aurora cluster endpoints (writer, reader, custom)"
status: active
severity: MEDIUM
triggers:
  - "endpoint not resolving"
  - "wrong endpoint"
  - "cluster endpoint"
  - "reader endpoint"
  - "custom endpoint"
  - "endpoint DNS"
owner: devops-agent
objective: "Identify and resolve Aurora endpoint configuration and resolution issues"
context: "Aurora provides multiple endpoint types: cluster (writer) endpoint, reader endpoint (DNS round-robin), instance endpoints, and custom endpoints. Misunderstanding endpoint behavior is a common source of issues."
---

## Phase 1 — Triage

MUST:
- List all cluster endpoints:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> \
    --query 'DBClusters[0].{Writer:Endpoint,Reader:ReaderEndpoint,CustomEndpoints:CustomEndpoints}'
  ```
- List all instance endpoints:
  ```
  aws rds describe-db-instances --filters Name=db-cluster-id,Values=<cluster-id> \
    --query 'DBInstances[].{Id:DBInstanceIdentifier,Endpoint:Endpoint.Address,Role:DBInstanceIdentifier}'
  ```
- Verify DNS resolution: `nslookup <cluster-endpoint>` or `dig <cluster-endpoint>`
- Check cluster status: `aws rds describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].Status'`

SHOULD:
- Check custom endpoints if configured:
  ```
  aws rds describe-db-cluster-endpoints --db-cluster-identifier <cluster-id>
  ```
- Verify which instance is the current writer:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> \
    --query 'DBClusters[0].DBClusterMembers[?IsClusterWriter==`true`].DBInstanceIdentifier'
  ```
- Test connectivity to each endpoint type separately

MAY:
- Check Route 53 if using custom DNS aliases
- Verify application connection string configuration

## Phase 2 — Remediate

MUST:
- For writer endpoint issues: verify the cluster has an active writer instance
- For reader endpoint not distributing: understand it is DNS round-robin, not a load balancer
- For custom endpoint issues: verify endpoint membership and type (READER, ANY)

SHOULD:
- Use RDS Proxy for better connection routing and failover handling
- Configure application to use cluster endpoint for writes and reader endpoint (or Proxy) for reads
- Set DNS TTL to 5 seconds or less in application/JVM settings

MAY:
- Create custom endpoints for specific workload routing (e.g., analytics readers)
- Implement application-level endpoint selection logic

## Common Issues

- symptoms: "Reader endpoint always resolves to the same instance"
  diagnosis: "DNS round-robin with caching. DNS resolver or application caches the IP."
  resolution: "Disable DNS caching, reduce TTL, or use RDS Proxy for connection-level distribution."

- symptoms: "Application connecting to reader for writes"
  diagnosis: "Application using reader endpoint or instance endpoint of a reader for write operations."
  resolution: "Use cluster (writer) endpoint for all write operations."

- symptoms: "Custom endpoint not including expected instances"
  diagnosis: "Custom endpoint membership is static — instances must be explicitly added."
  resolution: "Update custom endpoint membership: aws rds modify-db-cluster-endpoint."

## Safety Ratings
- GREEN: describe-db-clusters, describe-db-instances, describe-db-cluster-endpoints, nslookup, dig — read-only endpoint inspection
- YELLOW: modify-db-cluster-endpoint — recoverable endpoint configuration changes
- RED: delete-db-cluster, force-failover — destructive or high-impact operations in production

## Escalation Conditions
- "Database serves production traffic"
- "Fix requires modifying endpoint configuration affecting active connections"
- "Fix requires failover of Aurora cluster"
- "DNS propagation issues affecting application connectivity"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, endpoint addresses (expose network topology)
- HIGH: DNS resolution results (reveal internal infrastructure)
- MEDIUM: cluster membership, instance roles, custom endpoint configuration

## Prohibited Actions
- "NEVER suggest deleting a database cluster to fix endpoint issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest modifying production parameter groups without testing"
- "NEVER suggest force-failover in production without confirming application readiness"

## Phase 3 — Rollback
- "Restore from snapshot if endpoint configuration change causes issues"
- "Revert custom endpoint membership changes if routing is incorrect"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "endpoint_issue — <specific_cause>"
evidence:
  - type: cluster_endpoints
    content: "<endpoint configuration>"
  - type: dns_resolution
    content: "<DNS lookup results>"
severity: MEDIUM
mitigation:
  immediate: "Correct endpoint usage in application"
  long_term: "Implement RDS Proxy or application-level endpoint routing"
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
