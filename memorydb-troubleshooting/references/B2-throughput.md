---
title: "B2 — Throughput Issues"
description: "Diagnose throughput bottlenecks on MemoryDB clusters"
status: active
severity: HIGH
triggers:
  - "throughput limit"
  - "ops per second"
  - "bandwidth limit"
  - "network throughput"
  - "command throughput"
owner: devops-agent
objective: "Identify and resolve throughput bottlenecks on MemoryDB"
context: "MemoryDB throughput is determined by node type (network bandwidth), shard count (parallelism), and command efficiency. Throughput bottlenecks can occur at the network, CPU, or memory level."
---

## Phase 1 — Triage

MUST:
- Check network throughput: `aws cloudwatch get-metric-statistics --namespace AWS/MemoryDB --metric-name NetworkBytesIn --dimensions Name=ClusterName,Value=<cluster-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check CPU utilization across nodes
- Check commands processed: `redis-cli --tls -h <endpoint> INFO stats` (look at instantaneous_ops_per_sec)
- Verify node type network bandwidth limits

SHOULD:
- Check per-shard throughput distribution
- Verify client connection count and distribution
- Check for large value operations consuming bandwidth
- Monitor NetworkBytesOut metric

MAY:
- Analyze command mix (read/write ratio, command types)
- Check for pipeline usage efficiency
- Review client-side throughput measurements

## Phase 2 — Remediate

MUST:
- Scale up node type for higher network bandwidth if network-bound
- Add shards for higher aggregate throughput if CPU-bound
- Optimize large value operations

SHOULD:
- Use pipelining to reduce round-trip overhead
- Distribute reads across replicas
- Compress large values before storing

MAY:
- Implement client-side batching
- Use hash data structures instead of many individual keys

## Common Issues

- symptoms: "Throughput plateaus despite low CPU"
  diagnosis: "Network bandwidth limit of the node type is reached."
  resolution: "Scale up to a node type with higher network bandwidth."

- symptoms: "Throughput limited on a single shard"
  diagnosis: "Hot shard receiving disproportionate traffic due to key distribution."
  resolution: "Reshard to distribute load. Review key naming for better hash slot distribution."

- symptoms: "High throughput but increasing latency"
  diagnosis: "Node is approaching CPU saturation from command processing."
  resolution: "Add shards to distribute command processing. Scale up node type for more CPU."

## Output Format

```yaml
root_cause: "throughput — <specific_cause>"
evidence:
  - type: network_metrics
    content: "<network bytes in/out, bandwidth utilization>"
  - type: cpu_metrics
    content: "<CPU utilization per node>"
  - type: commands
    content: "<ops per second and command distribution>"
severity: HIGH
mitigation:
  immediate: "Scale up node type or add shards"
  long_term: "Optimize data access patterns and monitor throughput"
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
