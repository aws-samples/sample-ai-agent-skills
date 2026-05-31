---
title: "I2 — Log Driver Issues"
description: "Diagnose missing or broken container logging in ECS"
status: active
severity: MEDIUM
triggers:
  - "No logs appearing"
  - "Log driver error"
  - "CloudWatch logs missing"
owner: devops-agent
objective: "Fix container logging configuration"
context: "ECS supports multiple log drivers: awslogs (CloudWatch), splunk, fluentd, etc. awslogs is most common. Requires execution role with logs:CreateLogStream and logs:PutLogEvents. Log group must exist or awslogs-create-group must be true."
---

## Phase 1 — Triage

MUST:
- Check task definition logConfiguration: driver, options (log group, region, stream prefix)
- Verify the CloudWatch log group exists (or awslogs-create-group=true)
- Check execution role has logs:CreateLogStream and logs:PutLogEvents permissions
- Verify the log group name and region are correct

SHOULD:
- Check if logs:CreateLogGroup is needed (if log group doesn't exist)
- Verify the container is actually producing output to stdout/stderr
- Check if the log driver is causing task startup failures

MAY:
- Check CloudWatch Logs for the log stream (format: prefix/container-name/task-id)
- Verify if a custom log driver (Firelens) is misconfigured

## Common Issues

- symptoms: "No logs in CloudWatch, task runs fine"
  diagnosis: "Log group doesn't exist and awslogs-create-group is not set."
  resolution: "Create the log group manually or set awslogs-create-group=true in logConfiguration options."

- symptoms: "Task fails to start with log driver error"
  diagnosis: "Execution role missing CloudWatch Logs permissions."
  resolution: "Add logs:CreateLogStream, logs:PutLogEvents (and logs:CreateLogGroup if needed) to execution role."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
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
3. If IAM roles were modified, revert the task definition to the previous revision with correct role ARNs
4. If log configuration was changed, create a new task definition revision with the original log driver settings
## Output Format

```yaml
root_cause: "log_driver — <detail>"
evidence:
  - type: task_definition
    content: "<log configuration and execution role permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix log group, permissions, or log driver configuration"
  long_term: "Standardize logging configuration across services"
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
