---
title: "B2 â€” EFS Latency Issues"
description: "Diagnose high latency on EFS operations"
status: active
severity: MEDIUM
triggers:
  - "high latency"
  - "slow file operations"
  - "metadata operations slow"
  - "EFS lag"
owner: devops-agent
objective: "Identify causes of high EFS latency and optimize performance"
context: "EFS latency depends on performance mode, operation type (data vs metadata), file system size, and network path. General Purpose mode has lower latency than Max I/O. Metadata operations are inherently slower on NFS."
---

## Phase 1 â€” Triage

MUST:
- Check performance mode: `aws efs describe-file-systems --file-system-id <fs-id> --query 'FileSystems[].PerformanceMode'`
- Check PercentIOLimit metric for General Purpose mode
- Check if files are in IA storage (IA has higher first-byte latency)
- Verify the instance is using the mount target in its own AZ

SHOULD:
- Check MetadataIOBytes metric for metadata-heavy workloads
- Compare latency across different operation types (read, write, metadata)
- Check if the workload involves many small files (metadata-intensive)

MAY:
- Profile the application's I/O pattern
- Check for NFS attribute caching settings

## Phase 2 â€” Remediate

MUST:
- Ensure instances use mount targets in their own AZ
- Move frequently accessed files out of IA storage class

SHOULD:
- Optimize application to reduce metadata operations (ls, stat, chmod)
- Use larger I/O sizes for data operations
- Consider NFS mount options: `noresvport,rsize=1048576,wsize=1048576`

MAY:
- Consider Max I/O mode for highly parallel workloads (requires new file system)
- Evaluate if EFS is the right storage for the workload pattern

## Common Issues

- symptoms: "ls command takes seconds on directories with many files"
  diagnosis: "NFS metadata operations are slow for large directories."
  resolution: "Reduce directory sizes. Use hierarchical directory structures."

- symptoms: "First access to files is slow, subsequent access is fast"
  diagnosis: "Files are in IA storage class. First access has higher latency."
  resolution: "Use lifecycle policy to keep frequently accessed files in Standard."

- symptoms: "Latency increases under load"
  diagnosis: "General Purpose mode hitting IO limit or cross-AZ access."
  resolution: "Check PercentIOLimit. Ensure local AZ mount targets."

## Output Format

```yaml
root_cause: "high_latency â€” <specific_cause>"
evidence:
  - type: performance_mode
    content: "<performance mode and IO limit>"
  - type: latency_metrics
    content: "<latency measurements>"
severity: MEDIUM
mitigation:
  immediate: "Optimize mount configuration and AZ alignment"
  long_term: "Right-size performance mode and optimize I/O patterns"
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
