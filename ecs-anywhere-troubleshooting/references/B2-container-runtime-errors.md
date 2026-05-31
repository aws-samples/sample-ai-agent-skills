---
title: "B2 — Container Runtime Errors"
description: "Diagnose container runtime errors on external instances"
status: active
severity: HIGH
triggers: ["container failed to start", "Docker error", "containerd error", "runtime error", "CannotCreateContainerError"]
owner: devops-agent
objective: "Resolve container runtime errors on ECS Anywhere external instances"
context: "External instances require Docker or containerd pre-installed. Runtime errors prevent containers from starting or running correctly."
---

## Phase 1 — Triage

MUST:
- Check task stopped reason: `aws ecs describe-tasks --cluster <cluster-name> --tasks <task-arn> --query 'tasks[*].{Status:lastStatus,StoppedReason:stoppedReason,Containers:containers[*].{Name:name,Status:lastStatus,Reason:reason,ExitCode:exitCode}}'`
- Verify container runtime is running on the instance (via SSM): `aws ssm send-command --instance-ids <mi-id> --document-name AWS-RunShellScript --parameters commands=["systemctl status docker"]`
- Check ECS Agent status: `aws ecs describe-container-instances --cluster <cluster-name> --container-instances <instance-arn> --query 'containerInstances[*].{Agent:agentConnected,Status:status}'`

SHOULD:
- Check Docker/containerd logs on the instance for errors
- Verify disk space on the instance (container images require storage)
- Check if the container image exists and is accessible

MAY:
- Test pulling the container image manually on the instance
- Check for resource limits (ulimits, cgroups) that may affect containers
- Verify the container runtime version is compatible with the ECS Agent

## Phase 2 — Remediate

MUST:
- Restart Docker/containerd if stopped
- Fix disk space issues (prune unused images: `docker system prune`)
- Ensure container image is accessible from the instance

SHOULD:
- Update Docker/containerd to a supported version
- Configure log rotation for container logs to prevent disk exhaustion
- Verify container resource limits are appropriate

MAY:
- Switch from Docker to containerd (or vice versa) if runtime issues persist
- Implement container health checks in the task definition

## Common Issues

- symptoms: "Task stops with 'CannotPullContainerError'"
  diagnosis: "Instance cannot reach the container registry (ECR, Docker Hub)."
  resolution: "Verify outbound HTTPS to ECR endpoints. Check Docker proxy settings. Verify ECR authentication."

- symptoms: "Container starts but exits immediately with exit code 137"
  diagnosis: "Container killed by OOM (Out of Memory) — insufficient memory allocated."
  resolution: "Increase container memory in task definition. Check host memory availability."

## Safety Ratings

```
safety_ratings:
  - "ecs describe-tasks: GREEN — read-only"
  - "ecs describe-container-instances: GREEN — read-only"
  - "ssm send-command: RED — executes commands on remote instances"
  - "docker system prune: YELLOW — removes unused images and containers"
```

## Escalation Conditions

- Container runtime crashes repeatedly and cannot be stabilized
- Disk space cannot be freed without removing active container data
- Container image requires architecture not available on the external instance
- Runtime version incompatibility with ECS Agent cannot be resolved

## Data Sensitivity

- `aws ecs describe-tasks`: MEDIUM — reveals container configuration and environment
- `aws ssm send-command`: HIGH — executes commands on the instance
- Docker/containerd logs: HIGH — may contain application output and errors
- Container images: MEDIUM — may contain application code and configuration

## Prohibited Actions

- NEVER run docker system prune -a on production instances without verifying no active containers need cached images
- NEVER uninstall the container runtime on an active ECS instance
- NEVER modify container runtime configuration during active task execution
- NEVER ignore OOM kills — they indicate a resource planning issue

## Phase 3 — Rollback

1. If container runtime was updated, downgrade to previous version
2. If runtime configuration was changed, restore previous settings
3. Restart ECS Agent after runtime changes
4. Verify tasks can start and run correctly after rollback
5. Document the runtime issue and compatible versions

## Output Format

```yaml
root_cause: "container_runtime — <specific_cause>"
evidence:
  - type: runtime_error
    content: "<task stopped reason and runtime details>"
severity: HIGH
mitigation:
  immediate: "Fix runtime issue or restart"
  long_term: "Implement runtime monitoring and automated recovery"
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
  - command: "describe-tasks"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-services"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "execute-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest running containers as root to fix permission issues"
  - "NEVER suggest disabling task networking"
  - "NEVER remove IAM task role to troubleshoot"
