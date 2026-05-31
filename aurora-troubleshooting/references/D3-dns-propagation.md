---
title: "D3 — DNS Propagation"
description: "Diagnose DNS propagation issues after Aurora failover"
status: active
severity: HIGH
triggers:
  - "DNS propagation"
  - "DNS caching"
  - "stale DNS"
  - "old endpoint"
  - "connecting to old writer"
  - "DNS TTL"
owner: devops-agent
objective: "Identify and resolve DNS propagation issues affecting Aurora connectivity after failover"
context: "Aurora cluster endpoint DNS TTL is 5 seconds. After failover, the cluster endpoint updates to point to the new writer. However, DNS caching at the application, JVM, OS, or resolver level can cause connections to the old (now reader) instance. This is the most common cause of extended failover impact."
---

## Phase 1 — Triage

MUST:
- Verify current DNS resolution: `nslookup <cluster-endpoint>` or `dig <cluster-endpoint> +short`
- Compare resolved IP with current writer instance endpoint:
  ```
  aws rds describe-db-clusters --db-cluster-identifier <cluster-id> \
    --query 'DBClusters[0].DBClusterMembers[?IsClusterWriter==`true`].DBInstanceIdentifier'
  aws rds describe-db-instances --db-instance-identifier <writer-instance-id> \
    --query 'DBInstances[0].Endpoint.Address'
  ```
- Check if resolved IP matches the writer: `nslookup <writer-instance-endpoint>`
- Check DNS TTL: `dig <cluster-endpoint>` (look for TTL value in response)

SHOULD:
- Check application-level DNS caching:
  - Java/JVM: `networkaddress.cache.ttl` in `java.security` (default may be infinite)
  - Python: `socket.getaddrinfo` caching behavior
  - Node.js: DNS lookup caching
- Check OS-level DNS caching (systemd-resolved, nscd, dnsmasq)
- Verify the application is reconnecting after failover (not reusing stale connections)

MAY:
- Check Route 53 resolver if using custom DNS
- Review application connection pool eviction settings

## Phase 2 — Remediate

MUST:
- For JVM applications: set `networkaddress.cache.ttl=1` (or 0) in `java.security` or via `-Dsun.net.inetaddr.ttl=1`
- Implement connection retry logic with fresh DNS resolution on each retry
- Use RDS Proxy to avoid DNS propagation issues entirely (Proxy handles routing)

SHOULD:
- Disable OS-level DNS caching or set TTL to match Aurora's 5-second TTL
- Configure connection pools to validate connections before use (testOnBorrow)
- Set connection pool max lifetime to force periodic reconnection

MAY:
- Use Aurora JDBC/Python driver wrappers that handle failover-aware DNS resolution
- Implement application-level health checks that detect writer/reader role changes

## Common Issues

- symptoms: "Application still connecting to old writer 5+ minutes after failover"
  diagnosis: "JVM DNS caching with infinite TTL (Java default in some versions)."
  resolution: "Set networkaddress.cache.ttl=1 in java.security. Restart application."

- symptoms: "Some application instances recovered, others still on old writer"
  diagnosis: "Inconsistent DNS caching across application instances."
  resolution: "Standardize DNS TTL settings across all application instances."

- symptoms: "Writes failing with read-only error after failover"
  diagnosis: "Application connected to old writer which is now a reader."
  resolution: "Force DNS refresh. Implement connection validation. Use RDS Proxy."

## Safety Ratings
- GREEN: nslookup, dig, describe-db-clusters, describe-db-instances — read-only DNS and cluster inspection
- YELLOW: application DNS TTL configuration changes, connection pool settings — recoverable changes
- RED: force-failover, delete-db-cluster — destructive or high-impact operations in production

## Escalation Conditions
- "Database serves production traffic"
- "DNS propagation delay causing extended application downtime"
- "Fix requires application restarts to clear DNS cache"
- "Fix requires JVM security configuration changes"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, connection strings, application configuration (may contain passwords)
- HIGH: DNS resolution results and endpoint addresses (reveal internal infrastructure)
- MEDIUM: DNS TTL settings, JVM configuration, connection pool settings

## Prohibited Actions
- "NEVER suggest deleting a database cluster to fix DNS issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted cluster"
- "NEVER suggest force-failover in production without confirming application readiness"
- "NEVER suggest setting DNS TTL to 0 in production without understanding caching implications"

## Phase 3 — Rollback
- "If DNS configuration changes cause issues, revert TTL settings and restart applications"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"
- "Revert JVM security property changes if they cause unexpected behavior"

## Output Format

```yaml
root_cause: "dns_propagation — <specific_cause>"
evidence:
  - type: dns_resolution
    content: "<current DNS resolution vs expected>"
  - type: dns_ttl
    content: "<TTL settings at various levels>"
  - type: application_config
    content: "<DNS caching configuration>"
severity: HIGH
mitigation:
  immediate: "Force DNS refresh and reconnect applications"
  long_term: "Configure proper DNS TTL, use RDS Proxy, implement failover-aware drivers"
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
