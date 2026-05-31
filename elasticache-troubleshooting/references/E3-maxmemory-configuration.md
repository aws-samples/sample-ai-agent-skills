---
title: "E3 — ElastiCache maxmemory Configuration Issues"
description: "Diagnose maxmemory and maxmemory-reserved misconfiguration causing OOM or underutilization"
status: active
severity: HIGH
triggers:
  - "maxmemory"
  - "maxmemory-reserved"
  - "reserved-memory-percent"
  - "OOM"
  - "memory limit"
  - "memory configuration"
owner: devops-agent
objective: "Ensure maxmemory and reserved memory are correctly configured for the workload"
context: "maxmemory defines the total memory Redis can use. maxmemory-reserved (or reserved-memory-percent) sets aside memory for non-data overhead: replication buffers, client output buffers, Lua scripts, and internal data structures. AWS recommends 25% reservation. Too little reservation causes OOM during replication or heavy operations. Too much reservation wastes available cache memory. These are parameter group settings."
---

## Phase 1 — Triage

MUST:
- Check current maxmemory setting: `redis-cli -h <endpoint> -p 6379 CONFIG GET maxmemory`
- Check reserved memory: `aws elasticache describe-cache-parameters --cache-parameter-group-name <param-group> --query "Parameters[?ParameterName=='reserved-memory-percent']"`
- Check actual memory usage: `redis-cli -h <endpoint> -p 6379 INFO memory`
- Check eviction policy: `aws elasticache describe-cache-parameters --cache-parameter-group-name <param-group> --query "Parameters[?ParameterName=='maxmemory-policy']"`
- Check DatabaseMemoryUsagePercentage: `aws cloudwatch get-metric-statistics --namespace AWS/ElastiCache --metric-name DatabaseMemoryUsagePercentage --dimensions Name=CacheClusterId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`

SHOULD:
- Check if the parameter group is custom or default: `aws elasticache describe-cache-clusters --cache-cluster-id <cluster-id> --query 'CacheClusters[*].CacheParameterGroup'`
- Verify the node type's total memory to calculate effective data memory
- Check if replication is configured (replication buffers need reserved memory)
- Check for pub/sub usage (output buffers need reserved memory)

MAY:
- Check client output buffer limits: `redis-cli -h <endpoint> -p 6379 CONFIG GET client-output-buffer-limit`
- Review Lua script memory usage: `redis-cli -h <endpoint> -p 6379 INFO memory | grep used_memory_scripts`
- Check if the parameter group has pending changes requiring reboot

## Phase 2 — Remediate

MUST:
- Set reserved-memory-percent to 25 in the parameter group (AWS recommendation)
- Use a custom parameter group (do not modify the default parameter group)
- Apply parameter group changes and reboot if required: `aws elasticache modify-cache-parameter-group --cache-parameter-group-name <param-group> --parameter-name-values "ParameterName=reserved-memory-percent,ParameterValue=25"`

SHOULD:
- Increase reserved-memory-percent above 25% if using heavy pub/sub, Lua scripts, or many replicas
- Set appropriate eviction policy to match the workload
- Monitor memory usage after parameter changes

MAY:
- Adjust client-output-buffer-limit for pub/sub subscribers
- Consider scaling up if effective data memory is insufficient after proper reservation
- Document the parameter group configuration for the team

## Common Issues

- symptoms: "OOM errors during replication full sync"
  diagnosis: "reserved-memory-percent is too low. Replication buffer exceeds reserved memory."
  resolution: "Increase reserved-memory-percent to 25% or higher. Reboot nodes to apply."

- symptoms: "Low memory utilization despite sufficient data"
  diagnosis: "reserved-memory-percent is set too high, wasting available cache memory."
  resolution: "Reduce reserved-memory-percent to 25% (AWS default recommendation)."

- symptoms: "Parameter group change not taking effect"
  diagnosis: "reserved-memory-percent requires a reboot to apply."
  resolution: "Reboot the cache cluster to apply the parameter change. Check Apply Type in describe-cache-parameters."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Set reserved-memory-percent to 25% | YELLOW | Reduces available data memory; may trigger evictions; requires reboot |
| Use custom parameter group | GREEN | Best practice; no immediate impact |
| Adjust eviction policy | YELLOW | Changes eviction behavior; may evict different keys |
| Adjust client-output-buffer-limit | YELLOW | May disconnect pub/sub clients if set too low |
| Scale up node type | YELLOW | Requires failover for Redis with replication |

## Escalation Conditions

- OOM errors during replication full sync (reserved memory too low)
- Parameter group change requires reboot on a production cluster
- maxmemory misconfiguration causing data loss through evictions
- Multiple clusters sharing the same misconfigured parameter group
- Reserved memory change would push memory usage above eviction threshold

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `CONFIG GET maxmemory` | LOW | Configuration value only |
| `describe-cache-parameters` | LOW | Parameter configuration only |
| `INFO memory` | LOW | Memory statistics only |
| `get-metric-statistics` (DatabaseMemoryUsagePercentage) | LOW | Operational metrics only |

## Prohibited Actions

- NEVER suggest flushing all data (FLUSHALL/FLUSHDB) to fix maxmemory issues
- NEVER suggest disabling AUTH to reduce memory overhead
- NEVER suggest disabling encryption in transit to free memory
- NEVER suggest reducing node count during peak traffic
- NEVER set reserved-memory-percent below 15% (risks OOM during replication and background operations)

## Phase 3 — Rollback

If maxmemory configuration changes cause issues:
1. Revert reserved-memory-percent to previous value in parameter group: `aws elasticache modify-cache-parameter-group --cache-parameter-group-name <param-group> --parameter-name-values "ParameterName=reserved-memory-percent,ParameterValue=<previous>"`
2. Reboot nodes to apply parameter changes if required
3. If eviction policy was changed, revert to previous policy in parameter group
4. If client-output-buffer-limit was changed and disconnects clients, revert to previous values
5. Monitor DatabaseMemoryUsagePercentage and Evictions after rollback

## Output Format

```yaml
root_cause: "maxmemory_config — <specific_cause>"
evidence:
  - type: maxmemory
    content: "<maxmemory and reserved-memory-percent values>"
  - type: memory_usage
    content: "<INFO memory output>"
  - type: parameter_group
    content: "<parameter group configuration>"
severity: HIGH
mitigation:
  immediate: "Adjust reserved-memory-percent and reboot if required"
  long_term: "Use custom parameter groups with documented memory configuration"
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
