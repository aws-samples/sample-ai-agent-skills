---
title: "E1 â€” EFS IA Lifecycle Transitions"
description: "Diagnose lifecycle policy issues with Infrequent Access transitions"
status: active
severity: LOW
triggers:
  - "lifecycle policy"
  - "Infrequent Access"
  - "IA transition"
  - "storage class EFS"
owner: devops-agent
objective: "Identify why files are not transitioning to IA or transitioning unexpectedly"
context: "EFS lifecycle policies move files to Infrequent Access storage after a configured period of no access. Files must be at least 128 KB. IA storage has lower cost but higher per-access charges."
---

## Phase 1 â€” Triage

MUST:
- Check lifecycle configuration: `aws efs describe-lifecycle-configuration --file-system-id <fs-id>`
- Verify the transition period is set
- Check file sizes (files < 128 KB don't transition)

SHOULD:
- Check storage class distribution in CloudWatch metrics
- Verify files have not been accessed within the transition period

MAY:
- Check if transition-to-primary-storage-class is configured for automatic move-back

## Phase 2 â€” Remediate

MUST:
- Configure lifecycle policy if missing: `aws efs put-lifecycle-configuration --file-system-id <fs-id> --lifecycle-policies TransitionToIA=AFTER_30_DAYS`
- Understand that files < 128 KB will not transition

SHOULD:
- Configure transition back to Standard for frequently re-accessed files
- Monitor IA storage costs vs access charges

MAY:
- Adjust transition period based on access patterns

## Common Issues

- symptoms: "Files not moving to IA after configured period"
  diagnosis: "Files are smaller than 128 KB or have been accessed within the period."
  resolution: "Check file sizes. Files < 128 KB stay in Standard."

- symptoms: "High IA access charges"
  diagnosis: "Files in IA are being accessed frequently."
  resolution: "Enable transition back to Standard or increase the transition period."

- symptoms: "First access to IA files is slow"
  diagnosis: "IA storage has higher first-byte latency by design."
  resolution: "This is expected. Move frequently accessed files back to Standard."

## Output Format

```yaml
root_cause: "ia_transition_issue â€” <specific_cause>"
evidence:
  - type: lifecycle_config
    content: "<lifecycle policy>"
  - type: storage_metrics
    content: "<storage class distribution>"
severity: LOW
mitigation:
  immediate: "Adjust lifecycle policy"
  long_term: "Monitor storage class distribution and access patterns"
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
