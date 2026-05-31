---
title: "E2 â€” EFS Replication Issues"
description: "Diagnose EFS cross-region replication failures and lag"
status: active
severity: MEDIUM
triggers:
  - "replication"
  - "cross-region EFS"
  - "replication lag"
  - "disaster recovery EFS"
owner: devops-agent
objective: "Identify and fix EFS replication configuration and sync issues"
context: "EFS replication creates an asynchronous copy in another region. The destination is read-only. RPO is typically minutes. Replication is unidirectional."
---

## Phase 1 â€” Triage

MUST:
- Check replication configuration: `aws efs describe-replication-configurations --file-system-id <fs-id>`
- Verify replication status (Enabled, Error, Deleting)
- Check the destination file system status

SHOULD:
- Check TimeSinceLastSync metric in CloudWatch
- Verify both source and destination file systems are in Available state
- Check if the destination region has sufficient EFS quotas

MAY:
- Compare file system sizes between source and destination
- Check CloudTrail for replication configuration changes

## Phase 2 â€” Remediate

MUST:
- Fix replication errors by checking the error message in the configuration
- Ensure both file systems are in Available state
- Verify IAM permissions for replication

SHOULD:
- Monitor TimeSinceLastSync for RPO compliance
- Set up CloudWatch alarms for replication lag

MAY:
- Test failover procedure in a non-production environment

## Common Issues

- symptoms: "Replication status shows Error"
  diagnosis: "Destination file system or region has an issue."
  resolution: "Check the error message. Verify destination region quotas and file system state."

- symptoms: "High replication lag (TimeSinceLastSync increasing)"
  diagnosis: "High write rate on source exceeds replication throughput."
  resolution: "Monitor the metric. EFS replication is best-effort async."

- symptoms: "Cannot write to destination file system"
  diagnosis: "Destination is read-only by design during active replication."
  resolution: "This is expected. Perform failover to make destination writable."

## Output Format

```yaml
root_cause: "replication_issue â€” <specific_cause>"
evidence:
  - type: replication_config
    content: "<replication configuration and status>"
  - type: sync_metric
    content: "<TimeSinceLastSync>"
severity: MEDIUM
mitigation:
  immediate: "Fix replication errors"
  long_term: "Monitor replication lag and test failover procedures"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH — EFS provides shared storage for multiple compute instances |
| Data Risk | CRITICAL — File system contains application data and shared state |
| Recovery Complexity | HIGH — Data loss from file system deletion is unrecoverable without backups |
| Blast Radius | HIGH — Mount or performance issues affect all connected instances |

## Escalation Conditions

- File system unavailable or mount failures affecting multiple production instances
- Throughput degradation causing application timeouts across the fleet
- Burst credits depleted with no provisioned throughput configured
- Security group changes blocking NFS access for running workloads
- Replication lag exceeding RPO thresholds for disaster recovery
- IAM authorization failures preventing application access to file system

## Data Sensitivity

- File system contents may include application data, configuration files, and shared secrets
- Access point configurations reveal application directory structure and POSIX permissions
- Mount target IPs and DNS names are internal infrastructure identifiers
- Encryption keys (KMS) protect data at rest — key access must be tightly controlled
- EFS access logs may contain file paths revealing business logic or data organization

## PROHIBITED Actions

- NEVER suggest deleting a file system with active mount targets
- NEVER remove mount targets from availability zones with running instances
- NEVER change security group rules without verifying all connected instance groups
- NEVER disable encryption in transit on file systems handling sensitive data
- NEVER modify access point POSIX permissions without understanding application requirements
- NEVER switch throughput modes during peak usage without capacity planning

## Phase 3 — Rollback

- If mount target was removed: recreate in the same subnet with the same security groups
- If security group rules were changed: restore previous inbound TCP 2049 rules from backup
- If throughput mode was changed: revert to previous mode and monitor burst credit balance
- If access point permissions were modified: restore original POSIX UID/GID and directory path
- If lifecycle policy was changed: revert transition rules and verify IA storage class files
- Document all rollback actions with file system ID, mount target IDs, and timestamps
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

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
  - "NEVER suggest removing IAM authorization"
  - "NEVER suggest overly broad file system policies"
