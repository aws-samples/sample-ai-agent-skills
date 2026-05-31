---
title: "E1 — Vertical Scaling Issues"
description: "Diagnose node type change (vertical scaling) problems on MemoryDB"
status: active
severity: MEDIUM
triggers:
  - "scale up"
  - "scale down"
  - "node type change"
  - "instance type"
  - "vertical scaling"
owner: devops-agent
objective: "Identify and resolve vertical scaling issues on MemoryDB clusters"
context: "Vertical scaling changes the node type for all nodes in the cluster. The operation is online but causes brief failovers. Data is preserved through the transaction log. Scaling down may fail if the new node type has insufficient memory."
---

## Phase 1 — Triage

MUST:
- Describe cluster for current node type: `aws memorydb describe-clusters --cluster-name <cluster-name>`
- Check if a scaling operation is in progress (Status field)
- Verify the target node type is available: `aws memorydb describe-reserved-nodes-offerings`
- Check current memory and CPU utilization

SHOULD:
- Verify the target node type has sufficient memory for current data
- Check for ongoing operations that may conflict
- Review cluster events for scaling history: `aws memorydb describe-events --source-type cluster --duration 1440`
- Estimate scaling duration

MAY:
- Compare node type specifications (CPU, memory, network)
- Check for reserved node pricing
- Plan scaling during low-traffic periods

## Phase 2 — Remediate

MUST:
- Ensure target node type has enough memory for current dataset
- Plan for brief failovers during scaling (applications need retry logic)
- Monitor cluster health during and after scaling

SHOULD:
- Scale during maintenance windows or low-traffic periods
- Verify application handles brief disconnections
- Monitor performance after scaling to confirm improvement

MAY:
- Consider horizontal scaling (resharding) as an alternative
- Set up automated scaling based on metrics

## Common Issues

- symptoms: "Scaling fails — insufficient memory on target node type"
  diagnosis: "Current data size exceeds the memory of the target (smaller) node type."
  resolution: "Choose a node type with sufficient memory. Delete data or reshard before scaling down."

- symptoms: "Brief connection errors during scaling"
  diagnosis: "Scaling causes failovers as nodes are replaced — this is expected."
  resolution: "Implement retry logic in applications. Brief disconnections are normal during scaling."

- symptoms: "Scaling operation takes very long"
  diagnosis: "Large datasets require more time to sync to new nodes."
  resolution: "Wait for completion. Monitor progress via cluster events."

## Output Format

```yaml
root_cause: "vertical_scaling — <specific_cause>"
evidence:
  - type: cluster
    content: "<current node type and scaling status>"
  - type: metrics
    content: "<memory and CPU utilization>"
  - type: target
    content: "<target node type specifications>"
severity: MEDIUM
mitigation:
  immediate: "Fix scaling parameters or wait for completion"
  long_term: "Implement scaling procedures and application resilience"
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
