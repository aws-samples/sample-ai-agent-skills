---
title: "E2 — Performance Issues"
description: "Diagnose CloudShell performance degradation"
status: active
severity: MEDIUM
triggers: ["CloudShell slow", "commands take long", "lag", "unresponsive", "high latency"]
owner: devops-agent
objective: "Resolve CloudShell performance and responsiveness issues"
context: "CloudShell has limited CPU and memory resources. It is not designed for compute-intensive tasks. Performance varies by region and load."
---

## Phase 1 — Triage

MUST:
- Check system resources: `top -bn1 | head -20` (inside CloudShell)
- Check memory usage: `free -h` (inside CloudShell)
- Check disk I/O: `iostat -x 1 3` (inside CloudShell, if available)

SHOULD:
- Identify resource-heavy processes: `ps aux --sort=-%mem | head -10` (inside CloudShell)
- Check if storage is full (causes slowness): `df -h $HOME`

MAY:
- Test network latency to AWS services: `time aws sts get-caller-identity`
- Compare performance across regions by opening CloudShell in different regions

## Phase 2 — Remediate

MUST:
- Kill resource-intensive processes: `kill <pid>` (inside CloudShell)
- Free up storage if disk is full (I/O bottleneck)
- Close unused browser tabs running CloudShell sessions

SHOULD:
- Move compute-intensive tasks to EC2, Lambda, or ECS
- Use AWS CLI pagination and filters to reduce API response sizes

MAY:
- Optimize scripts to reduce memory usage
- Use lighter tools (e.g., jq instead of Python for JSON processing)

## Safety Ratings

```
safety_ratings:
  - "top / ps / free: GREEN — read-only system monitoring"
  - "kill <pid>: YELLOW — terminates a process, may lose unsaved work"
  - "Migrating to EC2: YELLOW — creates billable compute resources"
  - "Closing sessions: GREEN — no data loss if $HOME is backed up"
```

## Escalation Conditions

- Performance degradation affects all users in a region (possible service issue)
- CloudShell consistently runs out of memory for required operations
- Network latency to AWS APIs exceeds acceptable thresholds
- Performance issues coincide with AWS service events

## Data Sensitivity

- `top / ps aux`: LOW — process listing and resource usage
- `free -h`: LOW — memory statistics
- `aws sts get-caller-identity`: LOW — identity check
- Process output: varies — depends on what processes are running

## Prohibited Actions

- NEVER run cryptocurrency mining or resource-intensive computations in CloudShell
- NEVER attempt to modify CloudShell resource limits (not configurable)
- NEVER run stress tests or benchmarks that consume all available resources
- NEVER use CloudShell as a build server for large projects

## Phase 3 — Rollback

1. If a process was killed that was needed: restart it with appropriate resource limits
2. If files were deleted to free space: restore from S3 backup
3. If session was restarted: reconnect and verify environment state
4. If workload was migrated to EC2: verify EC2 instance is running correctly
5. Document performance baseline for future comparison

## Common Issues

- symptoms: "CloudShell becomes unresponsive when running npm install for a large project"
  diagnosis: "npm install exceeds CloudShell memory limits for large dependency trees."
  resolution: "Use EC2 or Cloud9 for large project builds. In CloudShell, use --production flag and limit dependencies."

## Output Format

```yaml
root_cause: "performance — <specific_cause>"
evidence:
  - type: performance_issue
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Kill resource-heavy processes and free resources"
  long_term: "Move compute-intensive workloads to appropriate AWS services"
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
