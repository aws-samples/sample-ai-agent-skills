---
title: "F2 — Restore Issues"
description: "Diagnose MemoryDB snapshot restore failures"
status: active
severity: HIGH
triggers:
  - "restore failed"
  - "restore from snapshot"
  - "restore error"
  - "cannot restore"
owner: devops-agent
objective: "Identify and resolve snapshot restore failures on MemoryDB"
context: "Restoring a MemoryDB snapshot creates a new cluster. Restore failures can be caused by subnet group issues, node type availability, ACL configuration, or KMS key access. MemoryDB snapshots can also restore to ElastiCache."
---

## Phase 1 — Triage

MUST:
- Describe the snapshot: `aws memorydb describe-snapshots --snapshot-name <snapshot-name>`
- Check the new cluster status: `aws memorydb describe-clusters --cluster-name <new-cluster-name>`
- Verify subnet group: `aws memorydb describe-subnet-groups --subnet-group-name <subnet-group>`
- Check ACL exists: `aws memorydb describe-acls --acl-name <acl-name>`

SHOULD:
- Verify node type is available in the target region/AZ
- Check KMS key accessibility
- Verify security group configuration
- Check CloudTrail for restore errors

MAY:
- Compare snapshot configuration with restore parameters
- Verify service quotas allow new cluster creation
- Check for region-specific limitations

## Phase 2 — Remediate

MUST:
- Fix the identified configuration issue
- Ensure ACL, subnet group, and security group are properly configured
- Retry restore after fixing the issue

SHOULD:
- Use compatible node type and shard configuration
- Update application connection strings for the new cluster
- Verify data integrity after restore

MAY:
- Test restore procedures periodically
- Document restore procedures for disaster recovery

## Common Issues

- symptoms: "Restore fails with ACL error"
  diagnosis: "The specified ACL does not exist or has no users."
  resolution: "Create the ACL and associate users before restoring."

- symptoms: "Restore creates cluster but nodes fail"
  diagnosis: "Node type not available or subnet group misconfigured."
  resolution: "Choose an available node type. Verify subnet group has subnets in multiple AZs."

- symptoms: "Want to restore MemoryDB snapshot to ElastiCache"
  diagnosis: "MemoryDB snapshots can be used to create ElastiCache clusters."
  resolution: "Use aws elasticache create-replication-group with --snapshot-name from MemoryDB."

## Output Format

```yaml
root_cause: "restore_issue — <specific_cause>"
evidence:
  - type: snapshot
    content: "<snapshot details>"
  - type: cluster
    content: "<restored cluster status>"
  - type: configuration
    content: "<subnet group, ACL, security group>"
severity: HIGH
mitigation:
  immediate: "Fix configuration and retry restore"
  long_term: "Document and test restore procedures"
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
