---
title: "I3 — Unexpected Instance Termination"
description: "Diagnose why an instance was unexpectedly terminated"
status: active
severity: CRITICAL
triggers:
  - "instance.*terminated.*unexpectedly"
  - "Client.InstanceInitiatedShutdownBehavior"
  - "Server.SpotInstanceTermination"
  - "Client.InternalError"
owner: devops-agent
objective: "Identify the termination cause and prevent recurrence"
context: "Instances can be terminated by: user API call, ASG scale-in, Spot interruption, instance-initiated shutdown behavior set to 'terminate', scheduled retirement, or internal errors. StateReason in describe-instances provides the cause."
---

## Phase 1 — Triage

MUST:
- Check termination reason: `aws ec2 describe-instances --instance-ids <id>` → StateReason
- Common StateReason codes:
  - `Client.UserInitiatedShutdown` — user or automation called TerminateInstances
  - `Client.InstanceInitiatedShutdownBehavior` — OS shutdown triggered termination
  - `Server.SpotInstanceTermination` — Spot instance reclaimed
  - `Server.InternalError` — AWS infrastructure issue
  - `Client.InternalError` — often EBS/KMS issue
- Check CloudTrail for TerminateInstances API call (who/what terminated it)

SHOULD:
- Check if instance was part of an ASG (scale-in event)
- Check Spot instance interruption notices (2-minute warning)
- Check instance-initiated-shutdown-behavior attribute
- Check for scheduled retirement events

MAY:
- Check AWS Health Dashboard for infrastructure events
- Review ASG scaling policies and activities

## Common Issues

- symptoms: "StateReason: Client.InstanceInitiatedShutdownBehavior"
  diagnosis: "OS issued a shutdown/halt command and the instance's shutdown behavior is set to 'terminate'."
  resolution: "Change shutdown behavior to 'stop': `aws ec2 modify-instance-attribute --instance-id <id> --instance-initiated-shutdown-behavior stop`"

- symptoms: "StateReason: Server.SpotInstanceTermination"
  diagnosis: "Spot instance reclaimed by AWS due to capacity needs or price change."
  resolution: "Use Spot Fleet with diversified allocation, or use On-Demand for critical workloads."

- symptoms: "CloudTrail shows TerminateInstances from an ASG"
  diagnosis: "ASG scale-in terminated the instance."
  resolution: "Review ASG scaling policies. Use instance protection for critical instances."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-instances StateReason: GREEN — read-only"
  - "CloudTrail TerminateInstances review: GREEN — read-only"
  - "Check ASG scaling activities: GREEN — read-only"
  - "Modify shutdown behavior to 'stop': YELLOW — changes instance lifecycle, recoverable"
  - "Enable instance protection in ASG: YELLOW — prevents scale-in termination, recoverable"
  - "Relaunch instance: YELLOW — new instance with new ID"
```

## Escalation Conditions
- Termination was caused by unauthorized API call (potential security incident)
- Multiple instances terminated simultaneously without explanation
- StateReason shows Server.InternalError (AWS infrastructure issue)
- Terminated instance had data on instance store volumes with no backup
- Spot interruption is affecting critical production workloads
- ASG scale-in terminated instances that should have been protected

## Data Sensitivity
- HIGH: CloudTrail TerminateInstances (reveals caller identity, source IP, assumed role)
- HIGH: describe-instances StateReason (reveals termination cause and context)
- MEDIUM: ASG scaling activities (reveals scaling policies and triggers)
- MEDIUM: Spot interruption notices (reveals capacity and pricing information)

## Prohibited Actions
- NEVER suggest ignoring unexpected terminations without investigating the cause
- NEVER suggest disabling ASG scaling policies as a permanent fix for unwanted terminations
- NEVER suggest using Spot instances for stateful workloads without interruption handling
- NEVER suggest setting all instances to termination-protected without understanding ASG implications

## Phase 3 — Rollback
- Terminated instances cannot be recovered — launch replacement from AMI/snapshot
- If shutdown behavior was changed: revert with `modify-instance-attribute --instance-initiated-shutdown-behavior terminate`
- If instance protection was enabled: disable if ASG needs to scale in normally
- If ASG scaling policy was modified: revert to previous scaling configuration
- If Spot Fleet was reconfigured: revert allocation strategy if new strategy is suboptimal

## Output Format

```yaml
root_cause: "<user_terminated|shutdown_behavior|spot_interruption|asg_scalein|internal_error>"
evidence:
  - type: state_reason
    content: "<StateReason from describe-instances>"
  - type: cloudtrail
    content: "<TerminateInstances event details>"
severity: CRITICAL
mitigation:
  immediate: "Relaunch instance or let ASG replace"
  long_term: "Fix shutdown behavior, use instance protection, diversify Spot fleet"
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
