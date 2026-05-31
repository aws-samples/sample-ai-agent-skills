---
title: "B1 â€” EFS Throughput Limits"
description: "Diagnose EFS throughput bottlenecks and mode selection"
status: active
severity: HIGH
triggers:
  - "slow throughput"
  - "throughput limit"
  - "EFS slow"
  - "read/write speed"
  - "PercentIOLimit"
owner: devops-agent
objective: "Identify throughput bottlenecks and recommend the correct throughput mode"
context: "EFS throughput depends on the throughput mode (Bursting, Provisioned, Elastic) and performance mode (General Purpose, Max I/O). Bursting throughput scales with file system size. Small file systems have limited baseline throughput."
---

## Phase 1 â€” Triage

MUST:
- Check file system throughput mode: `aws efs describe-file-systems --file-system-id <fs-id> --query 'FileSystems[].{ThroughputMode:ThroughputMode,ProvisionedThroughputInMibps:ProvisionedThroughputInMibps,SizeInBytes:SizeInBytes}'`
- Check PercentIOLimit: `aws cloudwatch get-metric-statistics --namespace AWS/EFS --metric-name PercentIOLimit --dimensions Name=FileSystemId,Value=<fs-id> --start-time <time> --end-time <time> --period 300 --statistics Average`
- Check TotalIOBytes: `aws cloudwatch get-metric-statistics --namespace AWS/EFS --metric-name TotalIOBytes --dimensions Name=FileSystemId,Value=<fs-id> --start-time <time> --end-time <time> --period 300 --statistics Sum`
- Check BurstCreditBalance if using Bursting mode

SHOULD:
- Calculate baseline throughput: file system size (TB) Ã— 50 MiB/s
- Check MeteredIOBytes for read vs write breakdown
- Compare actual throughput against the mode's limits

MAY:
- Check DataReadIOBytes and DataWriteIOBytes separately
- Review throughput patterns over the past week

## Phase 2 â€” Remediate

MUST:
- Switch to Provisioned throughput if burst credits are depleted and file system is small
- Switch to Elastic throughput for unpredictable workloads
- Increase provisioned throughput if already in Provisioned mode

SHOULD:
- Consider increasing file system size (add data) to increase bursting baseline
- Optimize application I/O patterns (larger reads/writes, fewer metadata operations)

MAY:
- Consider Max I/O performance mode if PercentIOLimit is consistently > 95%

## Common Issues

- symptoms: "Throughput drops to very low levels periodically"
  diagnosis: "Burst credits are depleted. File system is too small for sustained throughput."
  resolution: "Switch to Provisioned or Elastic throughput mode."

- symptoms: "PercentIOLimit consistently above 95%"
  diagnosis: "General Purpose mode is hitting its IOPS limit."
  resolution: "Consider Max I/O mode (requires new file system) or reduce metadata-heavy operations."

- symptoms: "Provisioned throughput not reaching configured level"
  diagnosis: "Application is not generating enough parallel I/O to saturate the throughput."
  resolution: "Use multiple threads/processes for parallel I/O. Check for client-side bottlenecks."

## Output Format

```yaml
root_cause: "throughput_limit â€” <specific_cause>"
evidence:
  - type: throughput_mode
    content: "<current throughput mode and settings>"
  - type: cloudwatch_metrics
    content: "<throughput and IO limit metrics>"
severity: HIGH
mitigation:
  immediate: "Switch throughput mode or increase provisioned throughput"
  long_term: "Right-size throughput mode based on workload patterns"
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
