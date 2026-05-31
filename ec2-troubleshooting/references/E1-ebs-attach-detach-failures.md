---
title: "E1 — EBS Attach/Detach Failures"
description: "Diagnose EBS volume attach or detach failures"
status: active
severity: HIGH
triggers:
  - "AttachVolume.*failed"
  - "volume.*attaching.*stuck"
  - "DetachVolume.*failed"
  - "volume.*busy"
  - "maximum.*volumes.*attached"
owner: devops-agent
objective: "Identify the attach/detach failure reason and restore volume access"
context: "EBS volumes must be in the same AZ as the instance. Attach limits depend on instance type (Nitro: 28 EBS + 2 NVMe, Xen: varies). Detach can fail if volume is in use by the OS. Force detach risks data corruption."
---

## Phase 1 — Triage

MUST:
- Check volume state: `aws ec2 describe-volumes --volume-ids <vol-id>` — state should be 'available' for attach
- Verify volume and instance are in the same AZ
- Check current attachment count vs instance limit: `aws ec2 describe-instance-types --instance-types <type>` → EbsInfo.MaximumAttachments
- For detach failures: check if volume is mounted in the OS

SHOULD:
- Check if volume is encrypted and instance role has KMS permissions
- For Nitro instances: check NVMe device mapping
- Check CloudTrail for AttachVolume/DetachVolume API errors

MAY:
- Check if volume is part of a RAID array or LVM group

## Common Issues

- symptoms: "AttachVolume fails with 'maximum number of volumes already attached'"
  diagnosis: "Instance has reached its EBS attachment limit."
  resolution: "Detach unused volumes. Check for ENIs consuming attachment slots (pre-Gen7 instances)."

- symptoms: "Volume stuck in 'attaching' state"
  diagnosis: "Underlying attachment issue. May be AZ mismatch or instance state issue."
  resolution: "Wait 10 minutes. If still stuck, force detach and retry. Check instance is running."

- symptoms: "DetachVolume fails, volume shows 'in-use'"
  diagnosis: "Volume is mounted in the OS. Must unmount before detaching."
  resolution: "Unmount the filesystem first (`umount`), then detach. Force detach as last resort (data corruption risk)."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-volumes, describe-instance-types: GREEN — read-only"
  - "describe-volumes-modifications: GREEN — read-only"
  - "Unmount filesystem before detach: YELLOW — makes data temporarily inaccessible"
  - "Detach volume (graceful): YELLOW — volume becomes available, re-attachable"
  - "Force detach volume: RED — risk of data corruption, filesystem inconsistency"
  - "Attach volume to different instance: YELLOW — changes device mapping, recoverable"
```

## Escalation Conditions
- Volume is stuck in 'attaching' or 'detaching' state for more than 10 minutes
- Force detach is the only option and volume contains critical data
- Volume is part of a RAID array or LVM group and cannot be safely detached
- Volume is encrypted with a KMS key the instance role cannot access
- Attachment limit reached and no volumes can be safely detached

## Data Sensitivity
- HIGH: describe-volumes (reveals encryption keys, snapshot IDs, attachment details)
- MEDIUM: CloudTrail AttachVolume/DetachVolume events (reveals volume operations history)
- LOW: describe-instance-types (public attachment limit data)

## Prohibited Actions
- NEVER suggest force-detaching a volume without warning about data corruption risk
- NEVER suggest detaching a root volume from a running instance
- NEVER suggest deleting a volume to free attachment slots without confirming data is backed up
- NEVER suggest attaching a volume across AZs (will fail — volumes are AZ-specific)

## Phase 3 — Rollback
- If volume was detached: re-attach to original instance with `attach-volume` using same device name
- If volume was force-detached: run fsck before mounting, restore from snapshot if filesystem is corrupt
- If volume was attached to wrong instance: detach and re-attach to correct instance
- If attachment slots were freed by detaching other volumes: re-attach those volumes after resolving the issue

## Output Format

```yaml
root_cause: "<az_mismatch|attachment_limit|volume_in_use|kms_permission|stuck_attaching>"
evidence:
  - type: volume_state
    content: "<describe-volumes output>"
severity: HIGH
mitigation:
  immediate: "Fix the specific blocker (unmount, free slots, fix AZ)"
  long_term: "Monitor attachment counts, use launch templates with correct AZ"
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
