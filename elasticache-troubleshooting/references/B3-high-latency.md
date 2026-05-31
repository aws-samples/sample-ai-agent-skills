---
title: "B3 — ElastiCache High Latency"
description: "Diagnose high latency on ElastiCache GET/SET operations and command execution"
status: active
severity: HIGH
triggers:
  - "high latency"
  - "slow response"
  - "p99 latency"
  - "response time"
  - "StringGetLatency"
  - "StringSetLatency"
owner: devops-agent
objective: "Identify the source of high latency and restore sub-millisecond response times"
context: "ElastiCache Redis typically delivers sub-millisecond latency for simple commands. High latency can originate from the network (client to node), the Redis engine (slow commands, CPU saturation), or the client (connection overhead, serialization). CloudWatch provides GetTypeCmdsLatency and SetTypeCmdsLatency metrics. Redis SLOWLOG captures commands exceeding the configured threshold."
---

## Phase 1 — Triage

MUST:
- Check latency metrics: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name GetTypeCmdsLatency --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Average,p99`
- Check SET latency: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name SetTypeCmdsLatency --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Average,p99`
- Check EngineCPU: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name EngineCPUUtilization --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Average,Maximum`
- Check slow log: `redis-cli -h <endpoint> -p 6379 SLOWLOG GET 25`
- Check network bytes: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name NetworkBytesIn --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Sum`

SHOULD:
- Check if large values are being transferred: `redis-cli -h <endpoint> -p 6379 --bigkeys`
- Verify client is in the same AZ as the node to minimize network latency
- Check if TLS is enabled (adds latency overhead): `aws elasticache describe-replication-groups --replication-group-id <repl-group-id> --query 'ReplicationGroups[*].TransitEncryptionEnabled'`
- Check for connection overhead (new connections vs pooled)

MAY:
- Use Redis LATENCY commands for detailed analysis: `redis-cli -h <endpoint> -p 6379 LATENCY LATEST`
- Check if background save is causing latency spikes: `redis-cli -h <endpoint> -p 6379 INFO persistence`
- Measure round-trip time: `redis-cli -h <endpoint> -p 6379 --latency`

## Phase 2 — Remediate

MUST:
- Optimize slow commands identified in SLOWLOG
- Scale up node type if EngineCPU exceeds 65%
- Use connection pooling to eliminate per-request connection overhead

SHOULD:
- Place clients in the same AZ as the ElastiCache node
- Use pipelining for batch operations to reduce round trips
- Compress large values before storing (application-level compression)

MAY:
- Enable Redis latency monitoring: `CONFIG SET latency-monitor-threshold 100`
- Consider cluster mode enabled to distribute load
- Evaluate if TLS overhead is acceptable for the latency requirements

## Common Issues

- symptoms: "p99 latency spikes to 50ms+ periodically"
  diagnosis: "Background save (BGSAVE) fork() causes latency spikes."
  resolution: "Schedule backups during off-peak. Ensure sufficient free memory for fork()."

- symptoms: "Consistent high latency on all commands"
  diagnosis: "EngineCPU saturated — single-threaded Redis cannot keep up."
  resolution: "Scale up node type. Offload reads to replicas. Add shards for write distribution."

- symptoms: "High latency only on specific commands"
  diagnosis: "Large key operations (HGETALL on 100K+ field hash, LRANGE on large list)."
  resolution: "Use HSCAN/SSCAN/LRANGE with pagination. Break large data structures into smaller ones."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Optimize slow commands in SLOWLOG | GREEN | Application-level improvement; immediate latency relief |
| Scale up node type | YELLOW | Requires failover for Redis with replication; brief downtime |
| Use connection pooling | GREEN | Application-level improvement; reduces connection overhead |
| Place clients in same AZ | GREEN | Infrastructure optimization; no data risk |
| Use pipelining for batch operations | GREEN | Application-level optimization |
| Enable latency monitoring | GREEN | Diagnostic configuration; minimal overhead |

## Escalation Conditions

- p99 latency exceeding 10ms on a production cluster (expected sub-millisecond)
- Latency spikes correlated with background save (BGSAVE) operations
- Consistent high latency across all commands (node saturation)
- Latency impacting customer-facing SLA
- TLS overhead causing unacceptable latency for latency-sensitive workloads

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `SLOWLOG GET` | MEDIUM | Exposes command patterns and key names |
| `--bigkeys` | MEDIUM | Exposes key names and sizes |
| `get-metric-statistics` (latency) | LOW | Operational metrics only |
| `LATENCY LATEST` | LOW | Latency event categories only |
| `--latency` (round-trip) | LOW | Network latency measurement only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to reduce latency
- NEVER suggest disabling AUTH to reduce command overhead
- NEVER suggest disabling encryption in transit without understanding compliance requirements
- NEVER suggest reducing node count during peak traffic
- NEVER run large O(N) commands to diagnose latency (they cause more latency)

## Phase 3 — Rollback

If latency remediation changes cause issues:
1. If node type was scaled up and causes issues, scale back to previous type
2. Revert application-level command optimizations if they cause functional issues
3. If connection pooling was implemented and causes connection leaks, revert to previous connection management
4. If pipelining was added and causes ordering issues, revert to sequential commands
5. Monitor GetTypeCmdsLatency and SetTypeCmdsLatency after rollback

## Output Format

```yaml
root_cause: "high_latency — <specific_cause>"
evidence:
  - type: latency_metrics
    content: "<GetTypeCmdsLatency and SetTypeCmdsLatency>"
  - type: engine_cpu
    content: "<EngineCPU utilization>"
  - type: slow_log
    content: "<slow commands>"
severity: HIGH
mitigation:
  immediate: "Optimize slow commands or scale up"
  long_term: "Implement pipelining, connection pooling, and AZ-aware routing"
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
