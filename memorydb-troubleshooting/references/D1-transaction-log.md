---
title: "D1 — Transaction Log Issues"
description: "Diagnose transaction log behavior and durability concerns on MemoryDB"
status: active
severity: HIGH
triggers:
  - "transaction log"
  - "durability"
  - "data loss"
  - "write acknowledgment"
  - "persistence"
owner: devops-agent
objective: "Understand and troubleshoot transaction log behavior and durability guarantees"
context: "MemoryDB's transaction log provides durability by writing data to a multi-AZ log before acknowledging writes. This ensures data survives node failures and restarts. Transaction log issues are rare but can manifest as write latency or durability concerns."
---

## Phase 1 — Triage

MUST:
- Verify cluster is healthy: `aws memorydb describe-clusters --cluster-name <cluster-name>`
- Check write latency: `aws cloudwatch get-metric-statistics --namespace AWS/MemoryDB --metric-name CommandLatency --dimensions Name=ClusterName,Value=<cluster-name> --start-time <start> --end-time <end> --period 60 --statistics p99`
- Verify multi-AZ deployment (always enabled for MemoryDB)
- Check for recent failover events: `aws memorydb describe-events --source-type cluster --duration 1440`

SHOULD:
- Compare write latency against baseline (single-digit ms is normal)
- Check for AZ-level issues in AWS Health Dashboard
- Verify data is present after a recent failover
- Monitor replication lag between primary and replicas

MAY:
- Review transaction log metrics if available
- Check for correlated network issues
- Verify data consistency across shards

## Phase 2 — Remediate

MUST:
- Verify data integrity after any failover event
- Ensure cluster has adequate replicas for failover
- Monitor write latency for anomalies

SHOULD:
- Understand that write latency includes transaction log write time
- Set appropriate client timeout values accounting for transaction log latency
- Document expected latency characteristics for the application

MAY:
- Compare MemoryDB write latency with application requirements
- Consider ElastiCache if sub-millisecond writes are required and durability is not

## Common Issues

- symptoms: "Write latency higher than ElastiCache"
  diagnosis: "This is expected — MemoryDB writes to the transaction log before acknowledging."
  resolution: "Single-digit ms write latency is normal. If sub-ms writes needed, evaluate ElastiCache."

- symptoms: "Data present after node failure"
  diagnosis: "This is expected — the transaction log preserves data across failures."
  resolution: "No action needed. This confirms MemoryDB durability is working correctly."

- symptoms: "Concern about data loss during AZ failure"
  diagnosis: "Transaction log is replicated across multiple AZs."
  resolution: "MemoryDB survives single-AZ failures. Data is preserved in the multi-AZ transaction log."

## Output Format

```yaml
root_cause: "transaction_log — <specific_cause>"
evidence:
  - type: cluster
    content: "<cluster health and multi-AZ status>"
  - type: latency
    content: "<write latency metrics>"
  - type: events
    content: "<failover and recovery events>"
severity: HIGH
mitigation:
  immediate: "Verify data integrity and cluster health"
  long_term: "Monitor write latency and understand durability guarantees"
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
