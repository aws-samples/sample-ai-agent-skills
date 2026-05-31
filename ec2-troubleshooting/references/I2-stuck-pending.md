---
title: "I2 — Instance Stuck in Pending State"
description: "Diagnose instances stuck in 'pending' state during launch"
status: active
severity: HIGH
triggers:
  - "stuck.*pending"
  - "instance.*pending.*long"
  - "launch.*timeout"
owner: devops-agent
objective: "Identify the launch blocker and get the instance to running state"
context: "Instances normally transition from pending to running in under 2 minutes. Stuck pending can be caused by EBS volume issues (encrypted volume KMS access, snapshot restore), ENI attachment issues, or host allocation problems."
---

## Phase 1 — Triage

MUST:
- Confirm instance state: `aws ec2 describe-instances --instance-ids <id>` → State, StateReason
- Check StateReason for specific error message
- Check if root volume is encrypted (KMS access needed during launch)
- Check CloudTrail for RunInstances event details

SHOULD:
- Check if the root volume is being restored from a large snapshot (can take time)
- Check for EBS volume attachment issues
- Verify the subnet has available IP addresses

## Phase 2 — Remediate

MUST:
- If KMS issue: fix key permissions and relaunch
- If snapshot restore: wait for restore to complete (large snapshots take longer)
- If stuck > 10 minutes with no progress: terminate and relaunch

SHOULD:
- Use fast snapshot restore (FSR) for large snapshots to avoid slow first-launch
- Pre-warm AMIs by launching and creating a new AMI

## Common Issues

- symptoms: "Instance pending for > 5 minutes, encrypted root volume"
  diagnosis: "KMS key access issue during volume attachment."
  resolution: "Check KMS key permissions. Terminate and relaunch with correct permissions."

- symptoms: "Instance pending, large AMI (100+ GB)"
  diagnosis: "EBS snapshot restore in progress. First launch from a snapshot is slower."
  resolution: "Wait for restore. Use Fast Snapshot Restore for future launches."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-instances StateReason: GREEN — read-only"
  - "CloudTrail event review: GREEN — read-only"
  - "Wait for snapshot restore: GREEN — no action required"
  - "Fix KMS permissions and relaunch: YELLOW — requires new instance"
  - "Terminate stuck-pending instance: RED — instance permanently destroyed"
  - "Enable Fast Snapshot Restore: YELLOW — cost increase, recoverable by disabling"
```

## Escalation Conditions
- Instance has been stuck in pending state for more than 10 minutes
- KMS key access issue requires cross-account coordination
- Stuck-pending instance is part of an ASG and blocking scaling operations
- Multiple instances are simultaneously stuck in pending state
- Root volume snapshot is very large (>1TB) and restore time is unpredictable

## Data Sensitivity
- MEDIUM: describe-instances StateReason (reveals launch failure details)
- MEDIUM: CloudTrail RunInstances event (reveals launch parameters, AMI, key pair)
- HIGH: Launch template user-data (may contain initialization scripts with secrets)

## Prohibited Actions
- NEVER suggest force-stopping a pending instance (not in a stoppable state)
- NEVER suggest modifying instance attributes while in pending state
- NEVER suggest launching additional instances if the issue is KMS-related (they'll also fail)
- NEVER suggest disabling EBS encryption to work around KMS issues

## Phase 3 — Rollback
- If stuck-pending instance was terminated: no rollback — launch a new instance with corrected configuration
- If KMS permissions were fixed: revert if permissions were too broad
- If Fast Snapshot Restore was enabled: disable with `disable-fast-snapshot-restores` to stop charges
- If new instance was launched: terminate if original instance eventually reaches running state

## Output Format

```yaml
root_cause: "<kms_access|snapshot_restore|eni_attachment|host_allocation> — <detail>"
evidence:
  - type: instance_state
    content: "<StateReason from describe-instances>"
severity: HIGH
mitigation:
  immediate: "Fix blocker or terminate and relaunch"
  long_term: "Use FSR, pre-warm AMIs, validate KMS permissions in launch templates"
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
