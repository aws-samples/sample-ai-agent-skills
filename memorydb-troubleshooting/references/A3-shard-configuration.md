---
title: "A3 — Shard Configuration Issues"
description: "Diagnose shard distribution and configuration problems"
status: active
severity: MEDIUM
triggers:
  - "shard configuration"
  - "hash slot"
  - "data distribution"
  - "uneven shards"
  - "shard count"
owner: devops-agent
objective: "Identify and resolve shard configuration and data distribution issues"
context: "MemoryDB distributes data across shards using hash slots (0-16383). Uneven data distribution, incorrect shard count, or hot keys can cause performance imbalances. Resharding is an online operation."
---

## Phase 1 — Triage

MUST:
- Describe cluster with shard details: `aws memorydb describe-clusters --cluster-name <cluster-name> --show-shard-details`
- Check hash slot distribution across shards
- Check per-shard memory usage via CloudWatch
- Verify shard count matches workload requirements

SHOULD:
- Check for hot keys causing uneven load: `redis-cli --tls -h <endpoint> CLUSTER INFO`
- Monitor per-shard CPU utilization
- Verify replica count per shard
- Check for CROSSSLOT errors in application logs

MAY:
- Analyze key distribution patterns
- Review hash tag usage for multi-key operations
- Check resharding history

## Phase 2 — Remediate

MUST:
- Reshard if data distribution is uneven or shards are overloaded
- Ensure shard count provides adequate memory and throughput capacity
- Verify application handles MOVED/ASK redirections correctly

SHOULD:
- Use hash tags for related keys that need multi-key operations
- Monitor per-shard metrics for early detection of imbalances
- Plan resharding during lower-traffic periods

MAY:
- Implement key naming conventions for better distribution
- Set up per-shard CloudWatch alarms

## Common Issues

- symptoms: "One shard has much higher memory usage than others"
  diagnosis: "Data distribution is uneven — hot keys or poor key naming causes hash slot imbalance."
  resolution: "Reshard to redistribute hash slots. Review key naming for better distribution."

- symptoms: "CROSSSLOT error on multi-key operations"
  diagnosis: "Multi-key commands (MGET, MSET, pipeline) operate on keys in different hash slots."
  resolution: "Use hash tags {tag} in key names to ensure related keys map to the same slot."

- symptoms: "Resharding takes very long"
  diagnosis: "Large data volume requires significant time to migrate between shards."
  resolution: "Resharding is online — it completes in the background. Monitor progress via cluster events."

## Output Format

```yaml
root_cause: "shard_configuration — <specific_cause>"
evidence:
  - type: shards
    content: "<shard details and hash slot distribution>"
  - type: metrics
    content: "<per-shard CPU and memory metrics>"
  - type: cluster_info
    content: "<redis-cli CLUSTER INFO output>"
severity: MEDIUM
mitigation:
  immediate: "Reshard or fix key distribution"
  long_term: "Implement key naming conventions and shard monitoring"
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
