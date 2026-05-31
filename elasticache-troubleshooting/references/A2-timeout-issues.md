---
title: "A2 — ElastiCache Timeout Issues"
description: "Diagnose connection timeouts and command timeouts on ElastiCache nodes"
status: active
severity: HIGH
triggers:
  - "timeout"
  - "read timeout"
  - "command timeout"
  - "socket timeout"
  - "connection timed out"
owner: devops-agent
objective: "Identify and resolve timeout issues between clients and ElastiCache nodes"
context: "Timeouts can occur at connection establishment or during command execution. Common causes include network latency, server overload (high CPU or memory pressure), slow commands blocking the event loop, maxclients reached, or client-side timeout configuration too aggressive. Redis is single-threaded for command execution — a single slow command blocks all others."
---

## Phase 1 — Triage

MUST:
- Check EngineCPU utilization: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name EngineCPUUtilization --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Average,Maximum`
- Check current connections vs limits: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name CurrConnections --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Check for slow commands: `redis-cli -h <endpoint> -p 6379 SLOWLOG GET 25`
- Check Redis INFO for blocked clients: `redis-cli -h <endpoint> -p 6379 INFO clients`
- Check network metrics: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name NetworkBytesIn --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Check the Redis timeout parameter: `aws elasticache describe-cache-parameters --cache-parameter-group-name <param-group> --query "Parameters[?ParameterName=='timeout']"`
- Check tcp-keepalive setting: `aws elasticache describe-cache-parameters --cache-parameter-group-name <param-group> --query "Parameters[?ParameterName=='tcp-keepalive']"`
- Review client-side timeout configuration (connect timeout, command timeout, socket timeout)
- Check for large key operations: `redis-cli -h <endpoint> -p 6379 --bigkeys`

MAY:
- Check latency history: `redis-cli -h <endpoint> -p 6379 LATENCY HISTORY command`
- Enable latency monitoring: `redis-cli -h <endpoint> -p 6379 CONFIG SET latency-monitor-threshold 100`
- Check if background save is running: `redis-cli -h <endpoint> -p 6379 INFO persistence`

## Phase 2 — Remediate

MUST:
- Identify and optimize slow commands (replace KEYS with SCAN, avoid large LRANGE/SMEMBERS/HGETALL)
- Increase client-side timeout values if they are too aggressive (recommend 1-5 seconds for command timeout)
- Scale up node type if EngineCPU consistently exceeds 65%

SHOULD:
- Set Redis timeout parameter to close idle connections (e.g., 300 seconds)
- Enable tcp-keepalive (recommended: 300 seconds) to detect dead connections
- Implement connection pooling with appropriate pool size

MAY:
- Enable Redis latency monitoring for ongoing analysis
- Consider cluster mode enabled to distribute load across shards
- Use read replicas to offload read traffic from the primary

## Common Issues

- symptoms: "Intermittent command timeouts during peak traffic"
  diagnosis: "EngineCPU exceeds 65% causing command queue delays."
  resolution: "Scale up node type or add read replicas. Optimize hot key access patterns."

- symptoms: "Timeout after KEYS or SMEMBERS command"
  diagnosis: "Slow O(N) command blocking the Redis event loop."
  resolution: "Replace KEYS with SCAN. Use SSCAN instead of SMEMBERS for large sets."

- symptoms: "Connection timeout but cluster shows available"
  diagnosis: "maxclients reached — new connections are refused."
  resolution: "Implement connection pooling. Scale up node type for higher maxclients. Close idle connections."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Optimize slow commands (KEYS → SCAN) | GREEN | Application-level improvement; no infrastructure risk |
| Increase client-side timeout values | GREEN | Application-level configuration change |
| Scale up node type | YELLOW | Requires failover for Redis with replication; brief downtime |
| Set Redis timeout parameter | GREEN | Closes idle connections; frees resources |
| Enable tcp-keepalive | GREEN | Detects dead connections; no data impact |
| Implement connection pooling | GREEN | Application-level improvement |

## Escalation Conditions

- Timeout issues affecting production application availability
- EngineCPU consistently above 80% causing command queue delays
- maxclients reached on a production cluster
- Slow commands blocking the Redis event loop for >1 second
- Timeout issues persisting after client-side timeout adjustments

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `SLOWLOG GET` | MEDIUM | Exposes command patterns and key names |
| `INFO clients` | LOW | Connection statistics only |
| `get-metric-statistics` (EngineCPU) | LOW | Operational metrics only |
| `--bigkeys` scan | MEDIUM | Exposes key names and sizes |
| `describe-cache-parameters` | LOW | Parameter configuration only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to fix timeout issues
- NEVER suggest disabling AUTH to reduce connection overhead
- NEVER suggest disabling encryption in transit to reduce TLS latency
- NEVER suggest reducing node count during peak traffic
- NEVER use KEYS command in production (causes the very timeouts being diagnosed)

## Phase 3 — Rollback

If timeout remediation changes cause issues:
1. Revert client-side timeout values to previous configuration
2. If Redis timeout parameter was set and disconnects active clients, set back to 0 (disabled): modify parameter group
3. If node type was scaled up and causes issues, scale back (note: causes another failover)
4. If connection pooling was implemented and causes connection leaks, revert to previous connection management
5. Verify timeout errors subside after rollback

## Output Format

```yaml
root_cause: "timeout — <specific_cause>"
evidence:
  - type: engine_cpu
    content: "<EngineCPU metrics>"
  - type: slow_log
    content: "<slow commands identified>"
  - type: connections
    content: "<current vs max connections>"
severity: HIGH
mitigation:
  immediate: "Optimize slow commands or increase client timeout values"
  long_term: "Implement connection pooling, scale appropriately, enable latency monitoring"
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
