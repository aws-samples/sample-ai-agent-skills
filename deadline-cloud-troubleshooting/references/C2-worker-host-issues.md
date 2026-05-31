---
title: "C2 — Worker Host Issues"
description: "Diagnose worker host (EC2 instance) infrastructure issues"
status: active
severity: HIGH
triggers: ["worker instance error", "instance terminated", "host resource exhaustion", "disk full", "out of memory"]
owner: devops-agent
objective: "Resolve worker host infrastructure issues"
context: "Worker hosts are EC2 instances (or on-premises machines for CMF) running the worker agent. Host issues include instance failures, resource exhaustion, and infrastructure problems separate from agent software issues."
---

## Phase 1 — Triage

MUST:
- Check instance status: `aws ec2 describe-instance-status --instance-ids <instance-id>`
- Check instance system and instance status checks
- Review CloudWatch metrics: `aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=InstanceId,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Check disk usage on the instance (render jobs can fill disks quickly)
- Verify memory usage (rendering is memory-intensive)
- Check for EC2 scheduled events or maintenance

MAY:
- Review instance system logs: `aws ec2 get-console-output --instance-id <instance-id>`
- Check EBS volume status for attached volumes
- Review Auto Scaling group events if applicable

## Phase 2 — Remediate

MUST:
- Replace terminated or failed instances (SMF does this automatically)
- Clear disk space if full (remove temporary render files)
- Fix resource exhaustion by right-sizing instance type

SHOULD:
- Configure instance monitoring for early warning
- Set up disk space alarms
- Use instance types with sufficient memory for render workloads

MAY:
- Configure instance store for temporary render data
- Set up EBS volume auto-expansion
- Implement render output cleanup scripts

## Safety Ratings

```
safety_ratings:
  - "describe-instance-status: GREEN — read-only instance check"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "get-console-output: GREEN — read-only system logs"
  - "terminate-instances: RED — kills instance and all running sessions"
  - "modify-instance-attribute: YELLOW — changes instance configuration"
```

## Escalation Conditions

- Multiple worker hosts failing simultaneously indicating infrastructure issue
- Instance type insufficient for workload requiring fleet reconfiguration
- EBS volume failures causing data loss
- Spot instance interruptions affecting production renders

## Data Sensitivity

- Instance metadata: LOW — instance IDs and types
- Console output: MEDIUM — may contain boot errors and system information
- CloudWatch metrics: LOW — aggregate performance data
- Disk contents: HIGH — may contain proprietary render assets and outputs

## Prohibited Actions

- NEVER terminate instances with running render sessions without saving progress
- NEVER resize instances without testing render software compatibility
- NEVER delete EBS volumes containing render output without backup
- NEVER ignore disk space warnings — render jobs will fail silently

## Phase 3 — Rollback

1. If instance was terminated, SMF will auto-replace; for CMF, launch replacement from AMI
2. If instance type was changed, revert to previous type and restart
3. If EBS volume was modified, restore from snapshot if needed
4. Verify worker agent re-registers on replacement instance

## Common Issues

- symptoms: "Worker host disk is 100% full"
  diagnosis: "Render output or temporary files filled the disk."
  resolution: "Clear temporary render files. Increase EBS volume size. Configure render output cleanup."

- symptoms: "Worker instance terminated unexpectedly"
  diagnosis: "Spot interruption, Auto Scaling scale-in, or instance health check failure."
  resolution: "Check termination reason in EC2 console. Use On-Demand for critical renders. Check ASG policies."

## Output Format

```yaml
root_cause: "worker_host — <specific_cause>"
evidence:
  - type: host_metrics
    content: "<instance status and resource utilization>"
severity: HIGH
mitigation:
  immediate: "Replace or fix host"
  long_term: "Implement host monitoring and auto-remediation"
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
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
