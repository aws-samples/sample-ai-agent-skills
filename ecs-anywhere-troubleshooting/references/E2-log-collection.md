---
title: "E2 — Log Collection"
description: "Diagnose log collection issues from ECS Anywhere external instances"
status: active
severity: MEDIUM
triggers: ["logs missing", "no container logs", "CloudWatch Logs empty", "log driver error"]
owner: devops-agent
objective: "Resolve log collection issues from ECS Anywhere containers"
context: "Container logs can be sent to CloudWatch Logs using the awslogs log driver. External instances need outbound connectivity to CloudWatch Logs endpoints."
---

## Phase 1 — Triage

MUST:
- Check task definition log configuration: `aws ecs describe-task-definition --task-definition <task-def> --query 'taskDefinition.containerDefinitions[*].{Name:name,LogDriver:logConfiguration.logDriver,Options:logConfiguration.options}'`
- Check if log group exists: `aws logs describe-log-groups --log-group-name-prefix <log-group-name>`
- Verify task execution role has CloudWatch Logs permissions

SHOULD:
- Check if the external instance can reach CloudWatch Logs endpoints
- Verify the log driver is supported (awslogs is recommended)
- Check for log delivery errors in ECS Agent logs

MAY:
- Test log delivery by running a simple task that outputs to stdout
- Check Docker logging configuration on the instance
- Verify log group retention settings

## Phase 2 — Remediate

MUST:
- Configure awslogs log driver in the task definition
- Create the log group if it doesn't exist (or enable auto-create in task execution role)
- Ensure task execution role has logs:CreateLogGroup, logs:CreateLogStream, logs:PutLogEvents

SHOULD:
- Set appropriate log retention to manage costs
- Configure log stream prefix for easy identification
- Verify outbound HTTPS to logs.<region>.amazonaws.com

MAY:
- Implement structured logging (JSON) for easier querying
- Set up CloudWatch Logs Insights queries for common patterns
- Configure log metric filters for alerting

## Common Issues

- symptoms: "Container logs not appearing in CloudWatch Logs"
  diagnosis: "awslogs log driver not configured, or task execution role missing logs permissions."
  resolution: "Add awslogs log driver to container definition. Ensure execution role has CloudWatch Logs permissions."

- symptoms: "Task fails with 'CannotStartContainerError: Error response from daemon: failed to initialize logging driver'"
  diagnosis: "Log group does not exist and auto-create is not enabled, or connectivity to CloudWatch Logs endpoint is blocked."
  resolution: "Create the log group manually or add logs:CreateLogGroup to the execution role. Verify network connectivity."

## Safety Ratings

```
safety_ratings:
  - "ecs describe-task-definition: GREEN — read-only"
  - "logs describe-log-groups: GREEN — read-only"
  - "logs create-log-group: YELLOW — creates new log group"
  - "ecs register-task-definition: YELLOW — creates new task definition version"
```

## Escalation Conditions

- Log delivery fails due to network restrictions that cannot be changed
- Log volume exceeds CloudWatch Logs quotas
- Sensitive data is being logged and requires redaction
- Log costs are excessive for the number of external instances

## Data Sensitivity

- `aws ecs describe-task-definition`: MEDIUM — reveals log configuration
- `aws logs describe-log-groups`: LOW — log group metadata
- Container logs: HIGH — may contain application data, errors, and PII
- Log configuration: LOW — log driver settings

## Prohibited Actions

- NEVER set log retention to unlimited without cost analysis
- NEVER log sensitive data (credentials, PII) in container output
- NEVER delete log groups containing production logs without backup
- NEVER disable logging on production containers

## Phase 3 — Rollback

1. If task definition log configuration was changed, revert to previous version
2. If log groups were created incorrectly, delete empty ones
3. If execution role was modified, restore previous permissions
4. Verify log collection works after rollback
5. Document the logging architecture and retention policies

## Output Format

```yaml
root_cause: "log_collection — <specific_cause>"
evidence:
  - type: logging_error
    content: "<log configuration and delivery details>"
severity: MEDIUM
mitigation:
  immediate: "Fix log driver configuration and permissions"
  long_term: "Implement centralized logging with retention policies"
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
