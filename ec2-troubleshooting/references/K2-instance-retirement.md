---
title: "K2 — Instance Retirement"
description: "Handle EC2 instance retirement due to hardware degradation"
status: active
severity: HIGH
triggers:
  - "instance-retirement"
  - "hardware.*degradation"
  - "retirement.*notification"
owner: devops-agent
objective: "Migrate the instance before retirement deadline"
context: "AWS retires instances when underlying hardware is irreparably degraded. EBS-backed instances can be stopped+started to migrate. Instance-store-backed instances must be terminated (data lost). Retirement notifications come via email and Personal Health Dashboard."
---

## Phase 1 — Triage

MUST:
- Confirm retirement event: `aws ec2 describe-instance-status --instance-ids <id>` → Events
- Check if instance is EBS-backed or instance-store-backed
- Note the retirement deadline date

## Phase 2 — Remediate

MUST:
- EBS-backed: stop+start to migrate to new host (preserves EBS data, changes public IP)
- Instance-store-backed: backup data, terminate, relaunch from AMI
- Do this BEFORE the retirement deadline

SHOULD:
- Set up EventBridge rule for retirement notifications
- Use ASG for automatic handling

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-instance-status Events: GREEN — read-only"
  - "describe-instances root device type: GREEN — read-only"
  - "Stop+start EBS-backed instance: YELLOW — changes public IP, brief downtime"
  - "Backup instance-store data: GREEN — non-destructive copy operation"
  - "Terminate instance-store-backed instance: RED — all instance store data permanently lost"
```

## Escalation Conditions
- Retirement deadline is imminent (< 24 hours) and migration hasn't been planned
- Instance is instance-store-backed with critical data that hasn't been backed up
- Instance is part of a cluster or distributed system requiring coordinated migration
- Stop+start fails due to capacity issues in the AZ
- Multiple instances are being retired simultaneously

## Data Sensitivity
- MEDIUM: describe-instance-status Events (reveals retirement schedule and deadline)
- MEDIUM: Personal Health Dashboard (reveals hardware degradation details)
- LOW: describe-instances root device type (reveals storage architecture)

## Prohibited Actions
- NEVER suggest ignoring retirement notifications (instance will be stopped by AWS)
- NEVER suggest terminating an instance-store-backed instance without first backing up data
- NEVER suggest contacting AWS to cancel retirement (hardware degradation cannot be reversed)
- NEVER suggest continuing to run on degraded hardware past the retirement deadline

## Phase 3 — Rollback
- If EBS-backed instance was stopped+started: no rollback needed (data preserved on EBS)
- If public IP changed: update DNS records or associate an Elastic IP
- If instance-store data was backed up and instance terminated: launch new instance and restore data
- If stop+start fails: try a different AZ or instance type, contact AWS Support

## Output Format

```yaml
root_cause: "Instance retirement — hardware degradation"
evidence:
  - type: retirement_event
    content: "<event details and deadline>"
severity: HIGH
mitigation:
  immediate: "Stop+start (EBS) or backup+terminate+relaunch (instance-store)"
  long_term: "EventBridge automation, ASG for auto-replacement"
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
