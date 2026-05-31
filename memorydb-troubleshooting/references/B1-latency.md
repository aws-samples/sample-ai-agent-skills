---
title: "B1 — Latency Issues"
description: "Diagnose high latency on MemoryDB operations"
status: active
severity: HIGH
triggers:
  - "high latency"
  - "slow response"
  - "latency spike"
  - "slow commands"
  - "response time"
owner: devops-agent
objective: "Identify and resolve latency issues on MemoryDB clusters"
context: "MemoryDB read latency is sub-millisecond (served from memory). Write latency is single-digit milliseconds due to the transaction log. Latency spikes can be caused by slow commands, network issues, memory pressure, or node resource exhaustion."
---

## Phase 1 — Triage

MUST:
- Check command latency: `aws cloudwatch get-metric-statistics --namespace AWS/MemoryDB --metric-name CommandLatency --dimensions Name=ClusterName,Value=<cluster-name> --start-time <start> --end-time <end> --period 60 --statistics Average`
- Check slow log: `redis-cli --tls -h <endpoint> SLOWLOG GET 25`
- Check CPU utilization: `aws cloudwatch get-metric-statistics --namespace AWS/MemoryDB --metric-name EngineCPUUtilization --dimensions Name=ClusterName,Value=<cluster-name> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Verify memory pressure: `aws cloudwatch get-metric-statistics --namespace AWS/MemoryDB --metric-name DatabaseMemoryUsagePercentage --dimensions Name=ClusterName,Value=<cluster-name> --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Identify slow commands from SLOWLOG (KEYS, SORT, SMEMBERS on large sets)
- Check network throughput utilization
- Verify client is in the same AZ/VPC as the cluster
- Check for connection storms or pool exhaustion

MAY:
- Analyze command distribution (read vs write ratio)
- Check for Lua script execution time
- Review client-side latency measurements

## Phase 2 — Remediate

MUST:
- Replace slow commands with efficient alternatives (SCAN instead of KEYS)
- Scale up node type if CPU or memory constrained
- Verify client network path to cluster

SHOULD:
- Optimize data structures for access patterns
- Use pipelining for batch operations
- Monitor latency percentiles (p99, p999)

MAY:
- Use read replicas for read-heavy workloads
- Implement client-side caching for frequently accessed keys

## Common Issues

- symptoms: "Write latency higher than expected (>5ms)"
  diagnosis: "MemoryDB transaction log adds write latency. This is expected behavior."
  resolution: "Single-digit ms write latency is normal for MemoryDB. Use ElastiCache if sub-ms writes needed."

- symptoms: "Latency spikes during specific operations"
  diagnosis: "Slow commands (KEYS *, SORT, large HGETALL) block the Redis event loop."
  resolution: "Replace KEYS with SCAN. Avoid large collection operations. Use HSCAN for large hashes."

- symptoms: "Latency increases with memory usage above 80%"
  diagnosis: "Memory pressure triggers eviction processing, adding latency to operations."
  resolution: "Scale up node type or add shards to distribute memory load. Keep usage below 80%."

## Output Format

```yaml
root_cause: "latency — <specific_cause>"
evidence:
  - type: latency_metrics
    content: "<command latency and percentiles>"
  - type: slowlog
    content: "<slow command entries>"
  - type: resources
    content: "<CPU, memory, network utilization>"
severity: HIGH
mitigation:
  immediate: "Fix slow commands and resource constraints"
  long_term: "Implement latency monitoring and command optimization"
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
