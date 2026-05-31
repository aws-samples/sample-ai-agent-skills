---
title: "E4 — Instance Store Data Loss"
description: "Diagnose and prevent instance store (ephemeral) data loss"
status: active
severity: CRITICAL
triggers:
  - "instance store.*lost"
  - "ephemeral.*data.*gone"
  - "NVMe.*missing"
  - "local disk.*empty"
owner: devops-agent
objective: "Understand the data loss cause and implement prevention measures"
context: "Instance store volumes are physically attached to the host. Data persists ONLY during the instance's running lifetime. Data is LOST on: stop, terminate, host failure, or hibernation. Data survives: reboot. Instance store is NOT backed up by AWS."
---

## Phase 1 — Triage

MUST:
- Confirm the instance type has instance store: `aws ec2 describe-instance-types --instance-types <type>` → InstanceStorageInfo
- Determine what event caused data loss: stop+start, termination, host failure, or scheduled maintenance
- Check if the instance was stopped (instance store data lost) or rebooted (data preserved)

SHOULD:
- Check CloudTrail for StopInstances or TerminateInstances events
- Check Personal Health Dashboard for host retirement or maintenance events
- Verify if the application was designed for ephemeral storage

MAY:
- Check if instance store volumes need to be reformatted after stop+start

## Phase 2 — Remediate

MUST:
- Accept that instance store data cannot be recovered after stop/terminate/host failure
- For future prevention: use EBS for persistent data, instance store only for temp/cache/scratch
- If data was critical: restore from application-level backups or replicas

SHOULD:
- Implement application-level replication for data on instance store
- Use instance store only for: caches, buffers, scratch data, temp files
- Configure applications to rebuild instance store data on boot

MAY:
- Consider using EBS instead of instance store for data that must persist
- Use RAID across instance store volumes for performance (not durability)

## Common Issues

- symptoms: "Data missing after stop+start"
  diagnosis: "Instance store data is always lost on stop. This is expected behavior, not a bug."
  resolution: "Use EBS for persistent data. Rebuild cache/temp data on start."

- symptoms: "Data missing after system status check failure and auto-recovery"
  diagnosis: "Auto-recovery migrates to new host. Instance store data is lost."
  resolution: "Design applications to handle instance store data loss gracefully."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-instance-types, describe-instances: GREEN — read-only"
  - "CloudTrail event review: GREEN — read-only"
  - "Restore from application-level backups: YELLOW — depends on backup integrity"
  - "Reconfigure application to use EBS: YELLOW — requires application changes and restart"
  - "Rebuild cache/temp data on instance store: GREEN — application-level operation"
```

## Escalation Conditions
- Critical data was stored only on instance store with no backups or replicas
- Instance store data loss was caused by unexpected host failure (not planned maintenance)
- Multiple instances lost instance store data simultaneously (potential AZ-level event)
- Application cannot rebuild instance store data automatically and requires manual intervention
- Data loss has compliance or regulatory implications

## Data Sensitivity
- HIGH: CloudTrail StopInstances/TerminateInstances events (reveals who initiated the action)
- MEDIUM: describe-instances (reveals instance lifecycle state and events)
- MEDIUM: Personal Health Dashboard events (reveals infrastructure events)

## Prohibited Actions
- NEVER suggest that instance store data can be recovered after stop/terminate/host failure
- NEVER suggest using instance store for databases, persistent state, or irreplaceable data
- NEVER suggest RAID 1 (mirroring) across instance store volumes as a durability solution (all lost together)
- NEVER suggest instance store as a replacement for EBS for any data that must survive instance lifecycle events

## Phase 3 — Rollback
- Instance store data loss is irreversible — there is no rollback
- If application was reconfigured to use EBS: revert application configuration to use instance store if EBS performance is insufficient
- If data was restored from backups: validate data integrity after restoration
- Prevention: implement application-level replication, use EBS for persistent data, automate cache rebuild on boot

## Output Format

```yaml
root_cause: "<stop_start|termination|host_failure|maintenance> — instance store data is ephemeral"
evidence:
  - type: event
    content: "<CloudTrail or Health event showing the trigger>"
severity: CRITICAL
mitigation:
  immediate: "Restore from backups or replicas"
  long_term: "Use EBS for persistent data, instance store only for ephemeral data"
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
