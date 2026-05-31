---
title: "A3 — ElastiCache DNS/Endpoint Resolution Issues"
description: "Diagnose DNS resolution failures and endpoint confusion for ElastiCache clusters"
status: active
severity: HIGH
triggers:
  - "DNS resolution"
  - "endpoint not found"
  - "could not resolve"
  - "wrong endpoint"
  - "NXDOMAIN"
  - "primary endpoint"
  - "reader endpoint"
  - "configuration endpoint"
owner: devops-agent
objective: "Resolve DNS and endpoint issues preventing clients from reaching ElastiCache nodes"
context: "ElastiCache provides multiple endpoint types: primary endpoint (Redis writes), reader endpoint (Redis reads), node endpoints (individual nodes), and configuration endpoint (Memcached auto-discovery). Using the wrong endpoint type causes unexpected behavior. DNS updates during failover take 30-60 seconds."
---

## Phase 1 — Triage

MUST:
- Get all endpoints for the replication group (Redis): `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].{Primary:NodeGroups[*].PrimaryEndpoint,Reader:NodeGroups[*].ReaderEndpoint}'`
- Get endpoints for cache cluster (Memcached): `aws elasticache describe-cache-clusters --cache-cluster-id <cluster-id> --show-cache-node-info --query 'CacheClusters[*].{ConfigEndpoint:ConfigurationEndpoint,Nodes:CacheNodes[*].Endpoint}'`
- Test DNS resolution: `nslookup <endpoint>`
- Verify the endpoint the application is using matches the correct endpoint type
- Check cluster status: `aws elasticache describe-cache-clusters --cache-cluster-id <cluster-id> --query 'CacheClusters[*].CacheClusterStatus'`

SHOULD:
- Check if a recent failover occurred: `aws elasticache describe-events --source-type replication-group --duration 60`
- Verify DNS TTL settings in the client (should respect low TTL for failover)
- Check if the application is caching DNS results beyond the TTL
- For cluster mode enabled, check all shard endpoints: `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].NodeGroups'`

MAY:
- Check VPC DNS settings (enableDnsHostnames, enableDnsSupport)
- Verify Route 53 resolver rules if using custom DNS
- Check for DNS caching in the application framework or JVM (Java: networkaddress.cache.ttl)

## Phase 2 — Remediate

MUST:
- Use the correct endpoint type: primary endpoint for writes, reader endpoint for reads (Redis); configuration endpoint for Memcached auto-discovery
- Ensure DNS TTL is respected by the client — do not cache DNS beyond the TTL
- After failover, wait 30-60 seconds for DNS propagation before assuming failure

SHOULD:
- Configure JVM DNS cache TTL to 60 seconds or less (networkaddress.cache.ttl=60)
- Use the ElastiCache Cluster Client for Memcached auto-discovery
- Implement connection retry logic that re-resolves DNS on connection failure

MAY:
- Use node endpoints for debugging but not for production traffic
- Set up CloudWatch alarms on failover events

## Common Issues

- symptoms: "Application connects to wrong node after failover"
  diagnosis: "Client is caching DNS beyond the TTL. After failover, the primary endpoint DNS changes but the client uses the stale IP."
  resolution: "Set DNS cache TTL to 60 seconds or less. For Java, set networkaddress.cache.ttl=60."

- symptoms: "NXDOMAIN when resolving ElastiCache endpoint"
  diagnosis: "VPC DNS resolution is disabled or the endpoint hostname is incorrect."
  resolution: "Enable enableDnsHostnames and enableDnsSupport on the VPC. Verify the endpoint from describe-replication-groups."

- symptoms: "Memcached client only sees some nodes"
  diagnosis: "Using individual node endpoints instead of the configuration endpoint for auto-discovery."
  resolution: "Use the configuration endpoint with the ElastiCache Cluster Client for automatic node discovery."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Use correct endpoint type | GREEN | Application-level configuration fix |
| Set DNS cache TTL to 60s | GREEN | Application/JVM configuration change |
| Implement connection retry with DNS re-resolution | GREEN | Application-level resilience |
| Configure JVM networkaddress.cache.ttl | GREEN | JVM configuration; no infrastructure change |

## Escalation Conditions

- DNS resolution failure preventing all client connections in production
- Failover completed but clients still connecting to old primary (stale DNS)
- VPC DNS resolution disabled affecting multiple services
- Endpoint confusion causing writes to reader endpoint or reads to wrong cluster
- DNS propagation delay exceeding 60 seconds after failover

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-replication-groups` (endpoints) | MEDIUM | Exposes cluster endpoints and architecture |
| `describe-cache-clusters` (endpoints) | MEDIUM | Exposes node endpoints |
| `nslookup` | LOW | DNS resolution only |
| `describe-events` | LOW | Operational events only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to fix DNS issues
- NEVER suggest disabling AUTH to troubleshoot endpoint resolution
- NEVER suggest disabling encryption in transit to simplify endpoint configuration
- NEVER suggest reducing node count during peak traffic
- NEVER use individual node endpoints for production traffic (use primary/reader/configuration endpoints)

## Phase 3 — Rollback

If DNS/endpoint changes cause issues:
1. Revert application endpoint configuration to previous value
2. If JVM DNS cache TTL was changed, revert networkaddress.cache.ttl to previous value
3. If connection retry logic was modified, revert to previous implementation
4. If VPC DNS settings were changed, revert enableDnsHostnames and enableDnsSupport
5. Verify DNS resolution and connectivity are restored after rollback

## Output Format

```yaml
root_cause: "dns_endpoint — <specific_cause>"
evidence:
  - type: endpoints
    content: "<primary, reader, and node endpoints>"
  - type: dns_resolution
    content: "<nslookup results>"
  - type: client_config
    content: "<endpoint used by application>"
severity: HIGH
mitigation:
  immediate: "Correct the endpoint used by the application"
  long_term: "Implement proper DNS TTL handling and connection retry logic"
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
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest disabling AUTH"
  - "NEVER suggest public subnet placement"
