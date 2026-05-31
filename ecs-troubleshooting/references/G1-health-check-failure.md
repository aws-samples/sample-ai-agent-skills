---
title: "G1 — Load Balancer Health Check Failure"
description: "Diagnose ECS tasks failing ALB/NLB health checks"
status: active
severity: HIGH
triggers:
  - "unhealthy target"
  - "Health check failed"
  - "Target deregistered"
  - "503 Service Unavailable"
owner: devops-agent
objective: "Fix load balancer health check failures and restore traffic flow"
context: "ELB health checks are separate from ECS container health checks. Both must pass. Health check path, port, protocol, and timing must match the application. Unhealthy targets are removed from rotation."
---

## Phase 1 — Triage

MUST:
- Check target health: `aws elbv2 describe-target-health --target-group-arn <arn>`
- Get health check configuration: `aws elbv2 describe-target-groups --target-group-arns <arn>`
- Verify health check path returns 200 (or configured success codes)
- Check if the application is listening on the health check port

SHOULD:
- Test the health check endpoint directly from within the VPC
- Check security group allows traffic from ALB to task on health check port
- Verify the health check timing (interval, timeout, healthy/unhealthy thresholds)

MAY:
- Check if the application takes longer to start than the health check grace period
- Verify the health check protocol matches (HTTP vs HTTPS)

## Common Issues

- symptoms: "All targets unhealthy, application is running"
  diagnosis: "Health check path returns non-200, or SG blocks ALB → task traffic."
  resolution: "Fix health check path to return 200, or add SG rule for ALB → task."

- symptoms: "Targets healthy briefly then become unhealthy"
  diagnosis: "Application crashes after initial startup, or health check endpoint becomes unresponsive under load."
  resolution: "Fix application stability, check for resource exhaustion."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Network configuration changes: YELLOW - May affect connectivity"
  - "Certificate/TLS changes: RED - May cause downtime if misconfigured"
```

## Escalation Conditions

- Service handles production traffic or is business-critical
- Fix requires modifying the task definition for a running service
- Resolution involves changing capacity provider or cluster configuration
- Multiple services affected suggesting cluster-level or account-level issue
- Deployment rollback has been triggered by the circuit breaker

## Data Sensitivity

HIGH - ECS task definitions may contain secrets ARNs, environment variables with credentials, and sensitive configuration. Container logs may capture application data including PII. Service Connect and service discovery configurations may expose internal architecture.

## Prohibited Actions

- NEVER suggest stopping all tasks in a service to fix a deployment issue - this causes a full outage
- NEVER suggest removing or modifying the task execution role while tasks are running
- NEVER recommend deleting a service without first scaling desired count to 0 and draining connections
- NEVER suggest modifying a running task definition - create a new revision instead
- NEVER recommend disabling the deployment circuit breaker on a production service

## Phase 3 - Rollback

1. If task definition was updated, roll back to previous revision: `aws ecs update-service --cluster <cluster> --service <service> --task-definition <previous-revision>`
2. If service desired count was changed, restore original: `aws ecs update-service --cluster <cluster> --service <service> --desired-count <original>`
3. If network configuration was changed, update service with original network settings: `aws ecs update-service --cluster <cluster> --service <service> --network-configuration <original>`
4. If health check settings were modified, restore original target group health check configuration
## Output Format

```yaml
root_cause: "lb_health_check — <detail>"
evidence:
  - type: target_group
    content: "<target health status and check configuration>"
severity: HIGH
mitigation:
  immediate: "Fix health check path/port or security group"
  long_term: "Implement dedicated health check endpoint, tune check parameters"
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
