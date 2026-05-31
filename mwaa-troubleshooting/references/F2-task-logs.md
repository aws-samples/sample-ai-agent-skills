---
title: "F2 — MWAA Task Log Issues"
description: "Diagnose missing or incomplete task logs in MWAA"
status: active
severity: MEDIUM
triggers:
  - "task logs missing"
  - "cannot view task log"
  - "log not found"
  - "task log empty"
  - "no log output"
owner: devops-agent
objective: "Identify why task logs are missing or incomplete and restore task log visibility"
context: "MWAA task logs are stored in the CloudWatch Task log group. Missing task logs can be caused by disabled logging, worker crashes, task timeouts, or log delivery delays. Task logs are essential for debugging task failures."
---

## Phase 1 — Triage

MUST:
- Check Task logging is enabled: `aws mwaa get-environment --name <env-name> --query 'Environment.LoggingConfiguration.TaskLogs'`
- Search for task logs: `aws logs filter-log-events --log-group-name airflow-<env-name>-Task --filter-pattern "<task_id>" --start-time <epoch-ms>`
- Check worker logs for task execution: `aws logs filter-log-events --log-group-name airflow-<env-name>-Worker --filter-pattern "<task_id>" --start-time <epoch-ms>`
- Verify task actually ran: use CLI token with `tasks states-for-dag-run <dag_id> <execution_date>`

SHOULD:
- Check Task log level (must be at least INFO to see standard output)
- Verify log stream naming pattern matches the task execution
- Check for worker crashes that may have lost buffered logs

MAY:
- Check if task was killed before logs could be flushed
- Verify CloudWatch Logs delivery latency

## Phase 2 — Remediate

MUST:
- Enable Task logging if disabled
- Set Task log level to at least INFO
- Verify execution role has CloudWatch Logs write permissions

SHOULD:
- Use explicit logging in task code (Python logging module) instead of print statements
- Set appropriate task timeouts to prevent indefinite hangs
- Monitor worker health to catch crashes early

MAY:
- Add structured logging to tasks for better searchability
- Create CloudWatch Insights queries for common task log patterns

## Common Issues

- symptoms: "Task log shows 'Log not found' in Airflow UI"
  diagnosis: "Task logging is disabled or log level is too restrictive."
  resolution: "Enable Task logging at INFO level in the environment configuration."

- symptoms: "Task log is truncated or incomplete"
  diagnosis: "Worker crashed or task was killed before logs were flushed."
  resolution: "Check worker logs for OOM or crash events. Increase worker resources if needed."

- symptoms: "Task log appears after a delay"
  diagnosis: "CloudWatch Logs delivery has inherent latency (seconds to minutes)."
  resolution: "Wait a few minutes and refresh. This is normal CloudWatch behavior."

## Output Format

```yaml
root_cause: "task_log_issue — <specific_cause>"
evidence:
  - type: task_logging_config
    content: "<enabled status and log level>"
  - type: task_state
    content: "<task execution state>"
  - type: worker_logs
    content: "<worker-level events for the task>"
severity: MEDIUM
mitigation:
  immediate: "Enable task logging and verify log delivery"
  long_term: "Implement structured logging in all tasks"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: MEDIUM — task logs may contain sensitive data processed by tasks
- Service Disruption Risk: LOW — task log issues do not affect task execution
- Compliance Impact: MEDIUM — missing task logs hinder debugging and audit

## Escalation Conditions
- Task logs completely missing for all tasks
- Task logs contain sensitive data that should not be logged
- Log retrieval from CloudWatch consistently fails
- Task log volume causes storage cost concerns
- Remote logging configuration breaks after environment update

## Data Sensitivity
- Task execution output — CONFIDENTIAL
- Error tracebacks — INTERNAL (may contain CONFIDENTIAL data)
- SQL queries in logs — CONFIDENTIAL
- API responses logged by tasks — CONFIDENTIAL
- Never expose database credentials or API keys found in task logs

## Prohibited Actions
- Do NOT disable task logging on production environments
- Do NOT grant public access to CloudWatch log groups containing task logs
- Do NOT ignore sensitive data exposure in task logs
- Do NOT delete task logs during active incident investigation
- Do NOT configure task logging to external destinations without encryption
- Do NOT log full API responses containing customer data

## Phase 3 — Rollback
- Restore remote logging configuration to previous settings
- Re-enable task logging if accidentally disabled
- Verify task logs appear in CloudWatch after configuration changes
- Restore log group permissions if access was inadvertently revoked
- Review and redact any sensitive data exposed in task logs
- Document task logging issues and update DAG coding guidelines

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest disabling webserver authentication"
  - "NEVER suggest public network access mode for production"
  - "NEVER suggest overly broad Airflow execution role"
