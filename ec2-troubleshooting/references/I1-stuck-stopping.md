---
title: "I1 — Instance Stuck in Stopping State"
description: "Diagnose instances stuck in 'stopping' state"
status: active
severity: HIGH
triggers:
  - "stuck.*stopping"
  - "instance.*stopping.*long"
  - "cannot stop"
owner: devops-agent
objective: "Resolve the stuck state and complete the stop operation"
context: "Instances can get stuck in 'stopping' when the underlying host has issues or the OS shutdown process hangs. AWS will eventually force-stop the instance, but this can take up to several hours."
---

## Phase 1 — Triage

MUST:
- Confirm instance state: `aws ec2 describe-instances --instance-ids <id>` → State
- Check how long the instance has been in 'stopping' state
- Check for scheduled maintenance events that may be related

SHOULD:
- Check system log for shutdown-related errors
- Check if the instance has instance store volumes (data will be lost)

## Phase 2 — Remediate

MUST:
- Wait — AWS will eventually force-stop the instance (can take up to several hours)
- If urgent: force stop is not available via API. Contact AWS Support for assistance.
- Do NOT try to terminate a stuck-stopping instance — it may also get stuck

SHOULD:
- Plan for data recovery if instance store volumes are involved
- Prepare a replacement instance

## Common Issues

- symptoms: "Instance in 'stopping' state for > 30 minutes"
  diagnosis: "OS shutdown process hung or underlying host issue."
  resolution: "Wait for AWS to force-stop. Contact AWS Support if urgent."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-instances to check state: GREEN — read-only"
  - "Check system log for shutdown errors: GREEN — read-only"
  - "Wait for AWS force-stop: GREEN — no action required"
  - "Contact AWS Support: GREEN — no infrastructure change"
  - "Prepare replacement instance: GREEN — no impact on stuck instance"
```

## Escalation Conditions
- Instance has been stuck in stopping state for more than 2 hours
- Instance has instance store volumes with unrecoverable data
- Stuck instance is blocking other operations (e.g., ASG replacement, deployment)
- Multiple instances are simultaneously stuck in stopping state
- Stuck instance is holding an Elastic IP or ENI needed by another instance

## Data Sensitivity
- MEDIUM: describe-instances (reveals instance state, lifecycle details)
- MEDIUM: system log (may contain shutdown process details)
- LOW: CloudTrail StopInstances event (reveals who initiated the stop)

## Prohibited Actions
- NEVER suggest terminating a stuck-stopping instance (may also get stuck)
- NEVER suggest force-stopping via API (not available for stuck-stopping state)
- NEVER suggest detaching volumes from a stuck-stopping instance
- NEVER suggest modifying instance attributes while in stopping state

## Phase 3 — Rollback
- Stuck-stopping state has no user-initiated rollback — AWS will eventually force-stop
- If instance store data was lost: restore from application-level backups or replicas
- If replacement instance was launched: terminate replacement if original instance recovers
- After force-stop completes: start the instance normally or launch replacement

## Output Format

```yaml
root_cause: "Instance stuck stopping — OS shutdown hung or host issue"
evidence:
  - type: instance_state
    content: "<describe-instances showing stopping state and duration>"
severity: HIGH
mitigation:
  immediate: "Wait for force-stop or contact AWS Support"
  long_term: "Configure shutdown scripts to timeout, use ASG for replacement"
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
