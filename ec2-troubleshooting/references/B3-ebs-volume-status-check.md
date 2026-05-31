---
title: "B3 — EBS Volume Status Check Failure"
description: "Diagnose EBS volume impairment and I/O issues"
status: active
severity: HIGH
triggers:
  - "volume status.*impaired"
  - "io-enabled.*insufficient-data"
  - "VolumeIOEnabled.*false"
owner: devops-agent
objective: "Identify the EBS volume issue and restore I/O or recover data"
context: "EBS volume status checks monitor I/O capability. Impaired volumes may have degraded performance or complete I/O failure. Causes include underlying storage infrastructure issues or volume corruption."
---

## Phase 1 — Triage

MUST:
- Check volume status: `aws ec2 describe-volume-status --volume-ids <vol-id>`
- Check volume details: `aws ec2 describe-volumes --volume-ids <vol-id>`
- Verify the volume is attached and the attachment state is 'attached'
- Check if the instance using this volume has status check failures

SHOULD:
- Check CloudWatch EBS metrics: VolumeReadOps, VolumeWriteOps, VolumeQueueLength, BurstBalance
- Check if auto-enable IO is configured on the volume

MAY:
- Check if other volumes on the same instance are affected
- Review AWS Health Dashboard for storage events

## Phase 2 — Remediate

MUST:
- If volume I/O is disabled: enable I/O to allow access (may have data inconsistency)
  `aws ec2 enable-volume-io --volume-id <vol-id>`
- If volume is impaired: create snapshot immediately for data recovery
- If root volume: stop instance, detach, create snapshot, create new volume from snapshot

SHOULD:
- Run filesystem check after enabling I/O (fsck for Linux, chkdsk for Windows)
- Monitor volume performance after recovery

MAY:
- Replace the volume with a new one created from the latest snapshot
- Consider migrating to io2 for higher durability (99.999%)

## Common Issues

- symptoms: "Volume status shows 'impaired', I/O disabled"
  diagnosis: "Underlying storage issue. Volume data may be inconsistent."
  resolution: "Enable I/O for access, snapshot immediately, run fsck, consider volume replacement."

- symptoms: "Volume queue length consistently high, latency increasing"
  diagnosis: "Volume IOPS or throughput limit reached. Not a status check failure but performance degradation."
  resolution: "See E3 (IOPS throttling runbook). Upgrade volume type or increase provisioned IOPS."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-volume-status, describe-volumes: GREEN — read-only diagnostics"
  - "enable-volume-io: YELLOW — allows I/O on potentially inconsistent volume, data risk"
  - "Create snapshot of impaired volume: GREEN — non-destructive backup operation"
  - "Stop instance to detach root volume: YELLOW — causes downtime, recoverable"
  - "Replace volume with new one from snapshot: YELLOW — requires careful data validation"
  - "Delete impaired volume after replacement: RED — permanent data loss if snapshot is incomplete"
```

## Escalation Conditions
- Volume contains database data or other write-ahead-log-dependent workloads
- Multiple volumes on the same instance are impaired simultaneously
- Volume is part of a RAID array or LVM group
- AWS Health Dashboard shows a regional storage event affecting multiple volumes
- Volume is encrypted with a customer-managed KMS key that may be compromised
- Impaired volume is the root volume of a production instance

## Data Sensitivity
- HIGH: get-console-output (may contain filesystem error details with file paths)
- HIGH: describe-volumes (reveals encryption keys, attachment details, account topology)
- MEDIUM: CloudWatch EBS metrics (reveals I/O patterns and workload characteristics)

## Prohibited Actions
- NEVER suggest deleting an impaired volume without first creating a snapshot
- NEVER suggest enabling I/O on a volume without warning about potential data inconsistency
- NEVER suggest running fsck on a mounted filesystem — must unmount first
- NEVER suggest force-detaching a volume while the instance is running unless absolutely necessary

## Phase 3 — Rollback
- If I/O was enabled on impaired volume: create snapshot immediately, then restore from last known-good snapshot if data is corrupt
- If volume was replaced: keep the old volume (do not delete) until new volume is verified healthy
- If instance was stopped to detach root volume: restart instance with original volume if recovery fails
- If filesystem check (fsck) made changes: restore from pre-fsck snapshot if filesystem is worse

## Output Format

```yaml
root_cause: "<volume_impaired|io_disabled|storage_event> — <detail>"
evidence:
  - type: volume_status
    content: "<describe-volume-status output>"
severity: HIGH
mitigation:
  immediate: "Enable I/O, snapshot for recovery"
  long_term: "Use io2 for critical volumes, automate snapshots"
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
