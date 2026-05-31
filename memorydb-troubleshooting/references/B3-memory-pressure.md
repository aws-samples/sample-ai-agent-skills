---
title: "B3 — Memory Pressure"
description: "Diagnose memory pressure, eviction, and OOM issues on MemoryDB"
status: active
severity: HIGH
triggers:
  - "memory pressure"
  - "eviction"
  - "OOM"
  - "out of memory"
  - "memory full"
  - "maxmemory"
owner: devops-agent
objective: "Identify and resolve memory pressure issues on MemoryDB clusters"
context: "MemoryDB stores all data in memory. When memory usage exceeds maxmemory, the eviction policy determines behavior. Evicted data is permanently removed (including from the transaction log). Memory pressure can cause latency spikes and data loss."
---

## Phase 1 — Triage

MUST:
- Check memory usage: `aws cloudwatch get-metric-statistics --namespace AWS/MemoryDB --metric-name DatabaseMemoryUsagePercentage --dimensions Name=ClusterName,Value=<cluster-name> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check evictions: `aws cloudwatch get-metric-statistics --namespace AWS/MemoryDB --metric-name Evictions --dimensions Name=ClusterName,Value=<cluster-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check memory info: `redis-cli --tls -h <endpoint> INFO memory`
- Verify eviction policy in parameter group

SHOULD:
- Check per-shard memory distribution
- Identify large keys: `redis-cli --tls -h <endpoint> --bigkeys`
- Check memory fragmentation ratio
- Monitor memory growth trend

MAY:
- Analyze key expiration patterns (TTL distribution)
- Check for memory leaks from client buffers
- Review Lua script memory usage

## Phase 2 — Remediate

MUST:
- Scale up node type for more memory or add shards to distribute data
- Set appropriate eviction policy (noeviction if data loss unacceptable)
- Delete unnecessary data or set TTLs on temporary data

SHOULD:
- Keep memory usage below 80% for headroom
- Implement TTL on all cache-like data
- Monitor eviction rate as an early warning

MAY:
- Optimize data structures for memory efficiency
- Use hash encoding for small objects
- Implement data archival for old data

## Common Issues

- symptoms: "Evictions occurring — data being lost"
  diagnosis: "Memory usage exceeded maxmemory and eviction policy is removing data."
  resolution: "Scale up node type or add shards. Set noeviction if data loss is unacceptable."

- symptoms: "OOM error on write operations"
  diagnosis: "Memory is full and eviction policy is noeviction — writes are rejected."
  resolution: "Scale up node type, add shards, or delete unnecessary data to free memory."

- symptoms: "Memory usage high but few keys stored"
  diagnosis: "Large values, memory fragmentation, or client output buffers consuming memory."
  resolution: "Check for large keys with --bigkeys. Monitor fragmentation ratio. Check client buffers."

## Output Format

```yaml
root_cause: "memory_pressure — <specific_cause>"
evidence:
  - type: memory_metrics
    content: "<memory usage percentage, evictions, fragmentation>"
  - type: memory_info
    content: "<redis-cli INFO memory output>"
  - type: large_keys
    content: "<big key analysis>"
severity: HIGH
mitigation:
  immediate: "Scale up or delete unnecessary data"
  long_term: "Implement memory monitoring, TTLs, and capacity planning"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete a MemoryDB cluster without verifying snapshot backups and data durability requirements
- **NEVER** modify ACL rules on production clusters without testing authentication changes in a staging environment
- **NEVER** force a failover on a production cluster during peak traffic without customer approval
- **NEVER** reduce the number of shards or replicas on a cluster serving production workloads without capacity analysis
- **NEVER** disable encryption in-transit or at-rest on clusters handling sensitive data

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
  - "NEVER suggest removing ACL authentication"
  - "NEVER suggest public subnet placement"
