---
title: "F1 — Snapshot Failures"
description: "Diagnose MemoryDB snapshot creation and management failures"
status: active
severity: HIGH
triggers:
  - "snapshot failed"
  - "snapshot error"
  - "backup failed"
  - "cannot create snapshot"
owner: devops-agent
objective: "Identify and resolve snapshot creation failures on MemoryDB"
context: "MemoryDB supports manual snapshots and automatic daily snapshots. Snapshot failures can be caused by KMS key issues, service quotas, cluster state, or concurrent operations."
---

## Phase 1 — Triage

MUST:
- List snapshots: `aws memorydb describe-snapshots --cluster-name <cluster-name>`
- Check snapshot status for failures
- Verify cluster is available: `aws memorydb describe-clusters --cluster-name <cluster-name>`
- Check KMS key status: `aws kms describe-key --key-id <key-id>`

SHOULD:
- Check cluster events for snapshot errors: `aws memorydb describe-events --source-type cluster --duration 1440`
- Verify automatic snapshot configuration (retention, window)
- Check service quotas for snapshot count
- Review CloudTrail for snapshot API errors

MAY:
- Check for concurrent snapshot operations
- Monitor snapshot creation duration
- Verify snapshot storage costs

## Phase 2 — Remediate

MUST:
- Fix KMS key access if that is the cause
- Ensure cluster is in available state
- Verify snapshot retention is configured (SnapshotRetentionLimit > 0)

SHOULD:
- Set snapshot window during low-traffic periods
- Monitor snapshot success with CloudWatch
- Delete old snapshots to stay within quotas

MAY:
- Automate snapshot management
- Set up alerts for snapshot failures

## Common Issues

- symptoms: "Automatic snapshots not being created"
  diagnosis: "SnapshotRetentionLimit is set to 0 (disabled)."
  resolution: "Set SnapshotRetentionLimit > 0 and configure SnapshotWindow."

- symptoms: "Snapshot creation fails with quota error"
  diagnosis: "Maximum number of manual snapshots reached."
  resolution: "Delete old manual snapshots to free quota."

- symptoms: "Snapshot takes very long for large cluster"
  diagnosis: "Snapshot size is proportional to data volume across all shards."
  resolution: "This is expected. Schedule snapshots during low-activity periods."

## Output Format

```yaml
root_cause: "snapshot_failure — <specific_cause>"
evidence:
  - type: snapshots
    content: "<snapshot list and status>"
  - type: cluster
    content: "<cluster status and snapshot configuration>"
  - type: kms
    content: "<KMS key status>"
severity: HIGH
mitigation:
  immediate: "Fix the identified issue and retry snapshot"
  long_term: "Configure automatic snapshots and monitoring"
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
