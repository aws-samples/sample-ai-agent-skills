---
title: "A1 — Essential Container Exit"
description: "Diagnose task failures caused by essential container stopping"
status: active
severity: HIGH
triggers:
  - "Essential container in task exited"
  - "Task stopped"
  - "Container exit code"
owner: devops-agent
objective: "Identify why the essential container exited and restore task stability"
context: "When a container marked 'essential' stops, the entire task is stopped. Exit codes: 0=normal, 1=app error, 137=OOM/SIGKILL, 139=SIGSEGV, 143=SIGTERM. Check stoppedReason and container exit code."
---

## Phase 1 — Triage

MUST:
- Get stopped task details: `aws ecs describe-tasks --cluster <cluster> --tasks <task-arn>` → stoppedReason, containers[].exitCode
- Check container logs: `aws logs filter-log-events --log-group-name <log-group> --filter-pattern "ERROR"`
- Identify which container is marked essential in the task definition
- Map exit code to cause: 0=clean exit, 1=app error, 137=OOM/SIGKILL, 139=segfault, 143=SIGTERM

SHOULD:
- Check if the container is crashing repeatedly (CrashLoopBackOff equivalent)
- Review recent task definition changes
- Check service events for patterns: `aws ecs describe-services --cluster <cluster> --services <service>`

MAY:
- Use ECS Exec to debug a running instance of the container
- Check if the issue correlates with deployments or config changes

## Common Issues

- symptoms: "Exit code 1, application error in logs"
  diagnosis: "Application code error — unhandled exception, missing config, or dependency failure."
  resolution: "Fix the application error based on log output."

- symptoms: "Exit code 137, no OOM in logs"
  diagnosis: "Container exceeded hard memory limit (OOM kill by ECS) or was manually stopped."
  resolution: "Increase memory limit in task definition or fix memory leak."

- symptoms: "Exit code 143, task stopped during deployment"
  diagnosis: "SIGTERM sent during rolling deployment. Container didn't shut down within stopTimeout."
  resolution: "Implement graceful shutdown handling, increase stopTimeout if needed."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
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
3. If deployment is stuck, force a new deployment with the last known good task definition: `aws ecs update-service --cluster <cluster> --service <service> --task-definition <last-good> --force-new-deployment`
4. If health check settings were modified, restore original target group health check configuration
5. If log configuration was changed, create a new task definition revision with the original log driver settings
## Output Format

```yaml
root_cause: "essential_container_exit — <exit code and cause>"
evidence:
  - type: ecs_task
    content: "<stoppedReason, exit code, and log excerpt>"
severity: HIGH
mitigation:
  immediate: "Fix the root cause based on exit code and logs"
  long_term: "Add health checks, implement graceful shutdown, set appropriate memory limits"
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
