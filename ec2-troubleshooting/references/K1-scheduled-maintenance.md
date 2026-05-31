---
title: "K1 — Scheduled Maintenance Events"
description: "Handle AWS scheduled maintenance events for EC2 instances"
status: active
severity: MEDIUM
triggers:
  - "scheduled.*maintenance"
  - "instance-reboot"
  - "system-reboot"
  - "instance-retirement"
owner: devops-agent
objective: "Proactively handle maintenance events to minimize downtime"
context: "AWS schedules maintenance for host updates, security patches, or hardware degradation. Events include: instance-reboot, system-reboot, system-maintenance, instance-retirement, instance-stop. You can often reschedule or preempt by stopping+starting."
---

## Phase 1 — Triage

MUST:
- Check scheduled events: `aws ec2 describe-instance-status --instance-ids <id>` → Events
- Identify event type and scheduled date
- Determine impact: reboot (brief downtime) vs retirement (must migrate)

SHOULD:
- Check all instances for upcoming events: `aws ec2 describe-instance-status --filters Name=event.code,Values=instance-reboot,system-reboot,instance-retirement`
- Plan maintenance window

## Phase 2 — Remediate

MUST:
- For instance-reboot/system-reboot: stop+start before the scheduled date to preempt (migrates to new host)
- For instance-retirement: stop+start (EBS-backed) or terminate+relaunch (instance-store-backed)
- For system-maintenance: stop+start to migrate off the host

SHOULD:
- Automate maintenance handling with EventBridge rules
- Use ASG for automatic replacement of retired instances

## Common Issues

- symptoms: "Scheduled instance-retirement event"
  diagnosis: "Underlying hardware degrading. Instance must be migrated."
  resolution: "Stop+start (EBS-backed) to migrate to new host. Do this before the scheduled date."

- symptoms: "Scheduled system-reboot"
  diagnosis: "Host needs a reboot for maintenance. Brief downtime expected."
  resolution: "Stop+start before the date to preempt, or let AWS reboot at the scheduled time."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-instance-status Events: GREEN — read-only"
  - "describe-instance-status with filters for all events: GREEN — read-only"
  - "Stop+start to preempt maintenance: YELLOW — changes public IP, brief downtime, loses instance store data"
  - "Let AWS reboot at scheduled time: YELLOW — brief downtime at scheduled time"
  - "Terminate+relaunch instance-store-backed: RED — instance store data permanently lost"
```

## Escalation Conditions
- Maintenance window conflicts with business-critical operations
- Instance is instance-store-backed and contains data that hasn't been backed up
- Multiple instances have overlapping maintenance windows
- Maintenance event cannot be preempted by stop+start (rare system-maintenance events)
- Instance is part of a cluster that cannot tolerate any member going offline

## Data Sensitivity
- MEDIUM: describe-instance-status Events (reveals maintenance schedule and event types)
- LOW: Personal Health Dashboard notifications (reveals scheduled maintenance details)

## Prohibited Actions
- NEVER suggest ignoring retirement events (instance will be stopped/terminated by AWS)
- NEVER suggest stop+start for instance-store-backed instances without first backing up data
- NEVER suggest rescheduling maintenance indefinitely (AWS will eventually enforce it)
- NEVER suggest disabling maintenance notifications

## Phase 3 — Rollback
- If instance was stopped+started to preempt: no rollback needed (instance on new host)
- If public IP changed after stop+start: update DNS records or use Elastic IP
- If instance store data was lost during migration: restore from application-level backups
- If maintenance was preempted but issue persists: contact AWS Support

## Output Format

```yaml
root_cause: "Scheduled maintenance — <event_type>"
evidence:
  - type: scheduled_event
    content: "<event details from describe-instance-status>"
severity: MEDIUM
mitigation:
  immediate: "Stop+start to preempt maintenance"
  long_term: "Automate with EventBridge, use ASG for auto-replacement"
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
