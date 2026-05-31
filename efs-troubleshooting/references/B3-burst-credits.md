---
title: "B3 â€” EFS Burst Credit Depletion"
description: "Diagnose burst credit exhaustion and throughput drops"
status: active
severity: HIGH
triggers:
  - "burst credits"
  - "BurstCreditBalance"
  - "throughput dropped"
  - "slow after initial burst"
owner: devops-agent
objective: "Identify burst credit depletion and recommend throughput mode changes"
context: "In Bursting mode, EFS accumulates burst credits at 50 MiB/s per TB of storage. Credits allow bursting to 100 MiB/s. When credits are depleted, throughput drops to the baseline rate. Small file systems deplete credits quickly."
---

## Phase 1 â€” Triage

MUST:
- Check throughput mode: `aws efs describe-file-systems --file-system-id <fs-id> --query 'FileSystems[].ThroughputMode'`
- Check BurstCreditBalance: `aws cloudwatch get-metric-statistics --namespace AWS/EFS --metric-name BurstCreditBalance --dimensions Name=FileSystemId,Value=<fs-id> --start-time <time> --end-time <time> --period 3600 --statistics Minimum`
- Check file system size: `aws efs describe-file-systems --file-system-id <fs-id> --query 'FileSystems[].SizeInBytes.Value'`
- Calculate baseline throughput: size_in_TB Ã— 50 MiB/s

SHOULD:
- Check burst credit trend over the past week
- Check TotalIOBytes to understand throughput demand
- Compare demand against baseline throughput

MAY:
- Estimate time to credit depletion based on current usage pattern
- Check if workload is bursty or sustained

## Phase 2 â€” Remediate

MUST:
- Switch to Provisioned throughput if credits are depleted and workload is sustained: `aws efs update-file-system --file-system-id <fs-id> --throughput-mode provisioned --provisioned-throughput-in-mibps <value>`
- Switch to Elastic throughput for unpredictable workloads

SHOULD:
- Set CloudWatch alarm on BurstCreditBalance approaching zero
- Right-size provisioned throughput based on actual usage

MAY:
- Add dummy data to increase file system size and burst baseline (not recommended)

## Common Issues

- symptoms: "Throughput was fast initially but slowed down significantly"
  diagnosis: "Burst credits depleted. File system is now at baseline throughput."
  resolution: "Switch to Provisioned or Elastic throughput mode."

- symptoms: "BurstCreditBalance is zero"
  diagnosis: "Sustained throughput exceeds the baseline rate for the file system size."
  resolution: "Switch throughput mode. Baseline for 100 GB is only 5 MiB/s."

- symptoms: "Throughput fluctuates between fast and slow"
  diagnosis: "Credits accumulate during idle periods and deplete during active periods."
  resolution: "Switch to Provisioned throughput for consistent performance."

## Output Format

```yaml
root_cause: "burst_credit_depletion â€” <specific_cause>"
evidence:
  - type: burst_credits
    content: "<BurstCreditBalance metric>"
  - type: file_system_size
    content: "<size and calculated baseline>"
severity: HIGH
mitigation:
  immediate: "Switch to Provisioned or Elastic throughput"
  long_term: "Monitor burst credits and set alarms"
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
