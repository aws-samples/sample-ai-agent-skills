---
title: "A2 — Node Issues"
description: "Diagnose MemoryDB node health and failure problems"
status: active
severity: HIGH
triggers:
  - "node failed"
  - "node not available"
  - "node replacement"
  - "node health"
  - "node reboot"
owner: devops-agent
objective: "Identify and resolve MemoryDB node health issues"
context: "MemoryDB nodes can fail due to hardware issues, memory exhaustion, or service events. Automatic failover promotes a replica to primary. Node replacement is automatic. Data is preserved through the transaction log."
---

## Phase 1 — Triage

MUST:
- Describe cluster for node status: `aws memorydb describe-clusters --cluster-name <cluster-name> --show-shard-details`
- Check node events: `aws memorydb describe-events --source-type node --duration 1440`
- Check CPU and memory metrics per node: `aws cloudwatch get-metric-statistics --namespace AWS/MemoryDB --metric-name CPUUtilization --dimensions Name=ClusterName,Value=<cluster-name>,Name=NodeId,Value=<node-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Verify shard health and replica count

SHOULD:
- Check for recent failover events
- Monitor ReplicationLag for replica nodes
- Verify node type is appropriate for workload
- Check for pending service updates: `aws memorydb describe-service-updates`

MAY:
- Compare metrics across nodes in the same shard
- Check for hot key distribution causing uneven load
- Review node replacement history

## Phase 2 — Remediate

MUST:
- Allow automatic node replacement to complete
- Verify data integrity after node replacement (transaction log ensures durability)
- Monitor cluster health during and after replacement

SHOULD:
- Scale up node type if resource exhaustion caused the failure
- Ensure adequate replica count for failover capacity
- Apply pending service updates during maintenance windows

MAY:
- Set up CloudWatch alarms for node health metrics
- Implement application retry logic for failover events

## Common Issues

- symptoms: "Node in replacing status for extended period"
  diagnosis: "Node replacement is in progress — MemoryDB is provisioning a new node and syncing data."
  resolution: "Wait for replacement to complete. Data is preserved via transaction log."

- symptoms: "Primary node failed and replica promoted"
  diagnosis: "Automatic failover occurred. A replica was promoted to primary."
  resolution: "Verify application reconnected to the new primary. Check for brief write interruption."

- symptoms: "All nodes in a shard showing high memory"
  diagnosis: "Shard is overloaded — data distribution may be uneven or node type is undersized."
  resolution: "Reshard to distribute data across more shards or scale up node type."

## Output Format

```yaml
root_cause: "node_issue — <specific_cause>"
evidence:
  - type: cluster
    content: "<cluster and shard details>"
  - type: node_metrics
    content: "<CPU, memory, connections per node>"
  - type: events
    content: "<node events and failover history>"
severity: HIGH
mitigation:
  immediate: "Allow automatic recovery and verify cluster health"
  long_term: "Right-size nodes and ensure adequate replica count"
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
