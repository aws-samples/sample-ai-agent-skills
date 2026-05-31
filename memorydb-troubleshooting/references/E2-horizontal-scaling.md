---
title: "E2 — Horizontal Scaling (Resharding)"
description: "Diagnose resharding and shard count change issues on MemoryDB"
status: active
severity: MEDIUM
triggers:
  - "resharding"
  - "add shards"
  - "remove shards"
  - "horizontal scaling"
  - "shard migration"
owner: devops-agent
objective: "Identify and resolve horizontal scaling (resharding) issues"
context: "Resharding adds or removes shards to change cluster capacity. It is an online operation that migrates hash slots between shards. During resharding, multi-key operations on migrating slots may fail. Large datasets take longer to reshard."
---

## Phase 1 — Triage

MUST:
- Describe cluster for current shard count: `aws memorydb describe-clusters --cluster-name <cluster-name> --show-shard-details`
- Check if resharding is in progress (cluster Status)
- Verify the target shard count is within limits (max 500)
- Check cluster events for resharding progress: `aws memorydb describe-events --source-type cluster --duration 1440`

SHOULD:
- Monitor per-shard metrics during resharding
- Check for MOVED/ASK errors in application logs
- Verify application handles slot migration redirections
- Estimate resharding duration based on data size

MAY:
- Plan resharding during low-traffic periods
- Check for multi-key operations that may be affected
- Review hash slot distribution after resharding

## Phase 2 — Remediate

MUST:
- Ensure application uses cluster-aware client that handles MOVED/ASK
- Monitor cluster health during resharding
- Verify data integrity after resharding completes

SHOULD:
- Plan resharding during lower-traffic periods
- Monitor replication lag during resharding
- Verify even hash slot distribution after completion

MAY:
- Automate resharding based on capacity metrics
- Document resharding procedures

## Common Issues

- symptoms: "Resharding takes very long"
  diagnosis: "Large data volume requires significant time to migrate between shards."
  resolution: "Resharding is online — wait for completion. Monitor via cluster events."

- symptoms: "CROSSSLOT errors during resharding"
  diagnosis: "Multi-key operations on keys in migrating slots fail during resharding."
  resolution: "Use hash tags for related keys. Implement retry logic for transient CROSSSLOT errors."

- symptoms: "Cannot reduce shard count below data requirements"
  diagnosis: "Remaining shards would not have enough memory for the redistributed data."
  resolution: "Delete data or scale up node type before reducing shard count."

## Output Format

```yaml
root_cause: "horizontal_scaling — <specific_cause>"
evidence:
  - type: cluster
    content: "<shard count and resharding status>"
  - type: events
    content: "<resharding progress events>"
  - type: metrics
    content: "<per-shard metrics during resharding>"
severity: MEDIUM
mitigation:
  immediate: "Monitor resharding progress and handle transient errors"
  long_term: "Implement capacity planning and resharding automation"
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
