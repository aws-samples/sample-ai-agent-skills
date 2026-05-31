---
title: "D3 — Disk I/O Bottleneck"
description: "Diagnose disk I/O performance issues on EC2 instances"
status: active
severity: MEDIUM
triggers:
  - "VolumeQueueLength.*high"
  - "await.*high"
  - "iowait.*high"
  - "disk.*slow"
owner: devops-agent
objective: "Identify the I/O bottleneck and restore disk performance"
context: "Disk I/O bottlenecks can occur at the EBS volume level (IOPS/throughput limits), the instance level (EBS-optimized bandwidth), or the OS level (filesystem, I/O scheduler). gp2 volumes have burst credits that deplete under sustained load."
---

## Phase 1 — Triage

MUST:
- Check CloudWatch EBS metrics: VolumeReadOps, VolumeWriteOps, VolumeQueueLength, VolumeThroughputPercentage
- For gp2: check BurstBalance metric (0% = throttled to baseline 3 IOPS/GB)
- Check instance EBS bandwidth: `aws ec2 describe-instance-types --instance-types <type>` → EbsInfo
- If SSM available: `iostat -x 1 5`, `iotop -b -n 3`

SHOULD:
- Compare volume IOPS/throughput against provisioned limits
- Check if multiple volumes are competing for instance EBS bandwidth
- Identify the process generating the most I/O

MAY:
- Check filesystem fragmentation and mount options
- Review I/O scheduler configuration

## Common Issues

- symptoms: "gp2 BurstBalance at 0%, high latency"
  diagnosis: "gp2 burst credits exhausted. Volume throttled to baseline (3 IOPS/GB, min 100)."
  resolution: "Migrate to gp3 (3000 baseline IOPS regardless of size) or increase gp2 size for higher baseline."

- symptoms: "High IOPS but instance EBS bandwidth saturated"
  diagnosis: "Instance type EBS-optimized bandwidth is the bottleneck, not the volume."
  resolution: "Upgrade to instance type with higher EBS bandwidth."

- symptoms: "High iowait but low volume IOPS"
  diagnosis: "Filesystem or OS-level issue (fragmentation, sync writes, journaling)."
  resolution: "Check mount options (noatime, barrier=0 for non-critical), defragment, tune I/O scheduler."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "CloudWatch EBS metrics, describe-instance-types: GREEN — read-only"
  - "iostat, iotop via SSM: GREEN — read-only diagnostics"
  - "Migrate gp2 to gp3: YELLOW — online migration, brief performance impact during transition"
  - "Increase provisioned IOPS: YELLOW — cost increase, recoverable by reducing"
  - "Modify volume type/size: YELLOW — online modification, 6-hour cooldown between changes"
  - "Change mount options (noatime): YELLOW — requires remount, recoverable"
  - "Upgrade instance type for EBS bandwidth: YELLOW — requires stop+start"
```

## Escalation Conditions
- Volume is in a 6-hour cooldown period from a recent modification and cannot be changed
- I/O bottleneck is causing database corruption or data loss
- Multiple volumes on the same instance are simultaneously throttled
- Instance EBS bandwidth is the bottleneck and upgrade requires architecture changes
- I/O bottleneck is affecting a production database with active transactions

## Data Sensitivity
- HIGH: iostat/iotop output via SSM (reveals I/O patterns, process names, file access patterns)
- MEDIUM: CloudWatch EBS metrics (reveals volume performance characteristics and workload patterns)
- LOW: describe-instance-types (public instance capability data)

## Prohibited Actions
- NEVER suggest reducing volume size (EBS volumes cannot be shrunk, only grown)
- NEVER suggest changing volume type during the 6-hour cooldown period (will fail)
- NEVER suggest disabling EBS optimization on an instance to troubleshoot
- NEVER suggest using instance store volumes for persistent data without explicit warning about data loss

## Phase 3 — Rollback
- If volume was migrated from gp2 to gp3: cannot revert to gp2 directly, create new gp2 volume from snapshot
- If provisioned IOPS were increased: modify volume to reduce IOPS (subject to 6-hour cooldown)
- If instance type was upgraded: stop instance, change back to original type, restart
- If mount options were changed: remount with original options or reboot to restore /etc/fstab defaults

## Output Format

```yaml
root_cause: "<gp2_burst_depleted|volume_iops_limit|instance_bandwidth|filesystem> — <detail>"
evidence:
  - type: cloudwatch_metric
    content: "<EBS metrics showing bottleneck>"
severity: MEDIUM
mitigation:
  immediate: "Migrate to gp3 or increase provisioned IOPS"
  long_term: "Right-size volumes and instance types for I/O requirements"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
