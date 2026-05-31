---
title: "B1 — System Status Check Failure"
description: "Diagnose and remediate EC2 system status check failures caused by underlying host issues"
status: active
severity: CRITICAL
triggers:
  - "StatusCheckFailed_System"
  - "system status check.*fail"
  - "impaired.*system"
owner: devops-agent
objective: "Restore instance to healthy state by migrating off the impaired host"
context: "System status checks monitor the AWS infrastructure hosting the instance. Failures indicate problems with the physical host, network connectivity to the host, or host software/hardware. The customer CANNOT fix these from inside the instance."
---

## Phase 1 — Triage

MUST:
- Confirm system status check failure: `aws ec2 describe-instance-status --instance-ids <id>`
- Check if instance status check is also failing (compound failure)
- Check for scheduled maintenance events: `aws ec2 describe-instance-status --include-all-instances`
- Check instance state (running, stopping, etc.)

SHOULD:
- Check CloudWatch StatusCheckFailed_System metric for duration and pattern
- Check if other instances in the same AZ/placement group are affected
- Review system log for hardware errors: `aws ec2 get-console-output --instance-id <id>`

MAY:
- Check AWS Health Dashboard for AZ-level events
- Check Personal Health Dashboard for account-specific events

## Phase 2 — Remediate

MUST:
- Stop and start the instance (NOT reboot) to migrate to new host hardware
- WARNING: Stop+start changes the public IPv4 address unless an Elastic IP is attached
- WARNING: Instance store data is LOST on stop+start
- If instance is EBS-backed: stop → start is safe for data on EBS volumes
- If instance is instance-store-backed: CANNOT stop. Must terminate and relaunch.

SHOULD:
- Verify instance recovers after start (check status checks again)
- If auto-recovery is not configured, set it up: CloudWatch alarm → EC2 recover action

MAY:
- Enable EC2 auto-recovery for future incidents
- Consider using a placement group with spread strategy for HA

## Guardrails

escalation_conditions:
  - "Instance fails system status check again after stop+start"
  - "Instance is instance-store-backed and cannot be stopped"
  - "Multiple instances failing system status checks simultaneously (AZ issue)"

safety_ratings:
  - "describe-instance-status, get-console-output: GREEN (read-only)"
  - "Stop + Start instance: YELLOW — changes public IP, loses instance store data"
  - "Terminate instance-store instance: RED — all data lost"

## Common Issues

- symptoms: "StatusCheckFailed_System = 1, instance is running"
  diagnosis: "Underlying host hardware or network issue. Customer cannot fix from inside the instance."
  resolution: "Stop and start the instance to migrate to healthy host. Verify with describe-instance-status after start."

- symptoms: "System status check fails repeatedly after stop+start"
  diagnosis: "Rare: may indicate a persistent issue. Try launching a new instance from the same AMI."
  resolution: "Launch replacement instance. If using ASG, terminate the unhealthy instance and let ASG replace it."

- symptoms: "Instance-store-backed instance with system status failure"
  diagnosis: "Cannot stop instance-store instances. Data will be lost."
  resolution: "If data is critical, attempt to copy data via network before terminating. Launch replacement from AMI."

## Output Format

```yaml
root_cause: "System status check failure — underlying host issue"
evidence:
  - type: instance_status
    content: "<describe-instance-status output showing SystemStatus impaired>"
severity: CRITICAL
mitigation:
  immediate: "Stop and start instance to migrate to new host"
  long_term: "Enable auto-recovery, use EBS-backed instances, configure ASG"
```

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

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
