---
title: "I1 — Memcached Auto-Discovery Issues"
description: "Diagnose Memcached auto-discovery failures preventing clients from detecting cluster nodes"
status: active
severity: HIGH
triggers:
  - "auto-discovery"
  - "Memcached discovery"
  - "configuration endpoint"
  - "node list"
  - "Memcached client"
  - "cluster client"
owner: devops-agent
objective: "Resolve Memcached auto-discovery issues so clients automatically detect all cluster nodes"
context: "ElastiCache Memcached supports auto-discovery — the configuration endpoint returns the current list of cache nodes. This requires the ElastiCache Cluster Client (not standard memcached clients). Standard memcached clients do not support auto-discovery and must be manually configured with node endpoints. When nodes are added or removed, auto-discovery clients automatically update their node list."
---

## Phase 1 — Triage

MUST:
- Check configuration endpoint: `aws elasticache describe-cache-clusters --cache-cluster-id <cluster-id> --show-cache-node-info --query 'CacheClusters[*].ConfigurationEndpoint'`
- Check all node endpoints: `aws elasticache describe-cache-clusters --cache-cluster-id <cluster-id> --show-cache-node-info --query 'CacheClusters[*].CacheNodes[*].Endpoint'`
- Test configuration endpoint: `echo "config get cluster" | nc <config-endpoint> 11211`
- Verify the client library supports auto-discovery (ElastiCache Cluster Client)
- Check cluster status: `aws elasticache describe-cache-clusters --cache-cluster-id <cluster-id> --query 'CacheClusters[*].CacheClusterStatus'`

SHOULD:
- Check if nodes were recently added or removed: `aws elasticache describe-events --source-identifier <cluster-id> --source-type cache-cluster --duration 1440`
- Verify the client is connecting to the configuration endpoint (not individual node endpoints)
- Check security group allows port 11211 from the client
- Verify DNS resolution of the configuration endpoint

MAY:
- Check the auto-discovery config version: `echo "config get cluster" | nc <config-endpoint> 11211`
- Review client library version for auto-discovery support
- Check if the client has a stale node list cached

## Phase 2 — Remediate

MUST:
- Use the ElastiCache Cluster Client (Java, .NET, PHP) for auto-discovery
- Connect to the configuration endpoint, not individual node endpoints
- Ensure security group allows inbound on port 11211

SHOULD:
- Configure the client to refresh the node list periodically (default is usually 60 seconds)
- Handle node addition/removal gracefully in the application
- Use consistent hashing to minimize cache redistribution when nodes change

MAY:
- Implement health checks for individual nodes
- Set up CloudWatch alarms for node count changes
- Consider migrating to Redis if auto-discovery limitations are problematic

## Common Issues

- symptoms: "Client only sees some nodes, not all"
  diagnosis: "Using standard memcached client without auto-discovery. Manually configured with partial node list."
  resolution: "Switch to ElastiCache Cluster Client and use the configuration endpoint."

- symptoms: "Auto-discovery returns empty node list"
  diagnosis: "Cluster is in a transitional state (adding/removing nodes)."
  resolution: "Wait for the cluster to reach 'available' status. Check events for ongoing operations."

- symptoms: "Client fails to connect to configuration endpoint"
  diagnosis: "Security group does not allow inbound on port 11211 or DNS resolution fails."
  resolution: "Add inbound rule for port 11211. Verify DNS resolution of the configuration endpoint."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Switch to ElastiCache Cluster Client | GREEN | Application-level client library change |
| Use configuration endpoint | GREEN | Application-level endpoint change |
| Fix security group for port 11211 | GREEN | Network configuration; allows specific traffic |
| Configure client refresh interval | GREEN | Application-level configuration |

## Escalation Conditions

- Auto-discovery failure preventing clients from detecting all nodes in production
- Standard memcached client used in production (no auto-discovery support)
- Cluster in transitional state returning empty node list
- Security group blocking configuration endpoint access
- Node addition/removal not detected by clients

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-cache-clusters` (endpoints) | MEDIUM | Exposes cluster endpoints and node configuration |
| `config get cluster` (Memcached) | LOW | Node list only |
| `describe-events` | LOW | Operational events only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to fix auto-discovery issues
- NEVER suggest disabling AUTH (not applicable to Memcached, but do not suggest removing security controls)
- NEVER suggest disabling encryption in transit to simplify auto-discovery
- NEVER suggest reducing node count during peak traffic
- NEVER use individual node endpoints for production traffic with Memcached (use configuration endpoint)

## Phase 3 — Rollback

If auto-discovery changes cause issues:
1. Revert to previous client library if ElastiCache Cluster Client causes issues
2. If configuration endpoint was changed, revert to previous endpoint
3. If security group rules were modified, revert to previous rules
4. If client refresh interval was changed, revert to previous value
5. Verify all nodes are discovered and traffic is distributed after rollback

## Output Format

```yaml
root_cause: "memcached_auto_discovery — <specific_cause>"
evidence:
  - type: config_endpoint
    content: "<configuration endpoint details>"
  - type: node_list
    content: "<current node endpoints>"
  - type: client_config
    content: "<client library and endpoint used>"
severity: HIGH
mitigation:
  immediate: "Use ElastiCache Cluster Client with configuration endpoint"
  long_term: "Implement auto-discovery with consistent hashing and health checks"
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
