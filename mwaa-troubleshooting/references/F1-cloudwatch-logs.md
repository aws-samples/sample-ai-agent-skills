---
title: "F1 — MWAA CloudWatch Log Issues"
description: "Diagnose CloudWatch log delivery and configuration issues in MWAA"
status: active
severity: MEDIUM
triggers:
  - "logs missing"
  - "no CloudWatch logs"
  - "log group empty"
  - "logging not working"
  - "cannot find logs"
owner: devops-agent
objective: "Identify why MWAA logs are not appearing in CloudWatch and restore log delivery"
context: "MWAA creates separate CloudWatch log groups for Scheduler, Worker, WebServer, DAGProcessing, and Task components. Each can be independently enabled/disabled at different log levels. Missing logs may indicate disabled log groups or IAM permission issues."
---

## Phase 1 — Triage

MUST:
- Check logging configuration: `aws mwaa get-environment --name <env-name> --query 'Environment.LoggingConfiguration'`
- List MWAA log groups: `aws logs describe-log-groups --log-group-name-prefix airflow-<env-name>`
- Check if log groups have recent events: `aws logs describe-log-streams --log-group-name airflow-<env-name>-Scheduler --order-by LastEventTime --descending --limit 5`
- Verify execution role has CloudWatch Logs permissions: `aws iam list-attached-role-policies --role-name <execution-role-name>`
- Check for log delivery errors in CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=logs.amazonaws.com --max-results 10`

SHOULD:
- Verify each component log level (CRITICAL, ERROR, WARNING, INFO, DEBUG)
- Check if KMS encryption on log groups is blocking writes
- Verify log group retention policy: `aws logs describe-log-groups --log-group-name-prefix airflow-<env-name> --query 'logGroups[*].{Name:logGroupName,Retention:retentionInDays}'`

MAY:
- Check CloudWatch Logs service quotas
- Verify no resource policy is blocking log delivery

## Phase 2 — Remediate

MUST:
- Enable disabled log groups through environment update
- Grant `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents` to execution role
- Set appropriate log levels (INFO for general troubleshooting, DEBUG for detailed investigation)

SHOULD:
- Set log retention policies to manage costs
- Enable all component log groups at minimum ERROR level
- Grant KMS permissions if using encrypted log groups

MAY:
- Create CloudWatch metric filters for common error patterns
- Set up CloudWatch alarms on error log frequency

## Common Issues

- symptoms: "No log groups exist for the MWAA environment"
  diagnosis: "All logging is disabled in the environment configuration."
  resolution: "Enable logging for required components via environment update."

- symptoms: "Log groups exist but are empty"
  diagnosis: "Execution role lacks CloudWatch Logs permissions."
  resolution: "Add logs:CreateLogStream and logs:PutLogEvents to the execution role."

- symptoms: "Some component logs present but Task logs missing"
  diagnosis: "Task logging is disabled independently from other components."
  resolution: "Enable Task logging in the environment logging configuration."

## Output Format

```yaml
root_cause: "cloudwatch_log_issue — <specific_cause>"
evidence:
  - type: logging_config
    content: "<enabled/disabled status per component>"
  - type: log_groups
    content: "<existing log groups and last event times>"
  - type: iam_permissions
    content: "<execution role CloudWatch Logs permissions>"
severity: MEDIUM
mitigation:
  immediate: "Enable logging and fix IAM permissions"
  long_term: "Standardize logging configuration across all environments"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: MEDIUM — logs may contain sensitive operational data
- Service Disruption Risk: LOW — logging issues do not affect DAG execution
- Compliance Impact: MEDIUM — missing logs can affect audit and compliance requirements

## Escalation Conditions
- All MWAA log groups stop receiving logs
- Log delivery latency exceeds acceptable thresholds
- CloudWatch Logs permissions revoked from execution role
- Log volume causes unexpected cost increases
- Sensitive data discovered in log output

## Data Sensitivity
- Scheduler logs — INTERNAL
- Worker logs — INTERNAL (may contain CONFIDENTIAL task output)
- Web server logs — INTERNAL
- DAG processing logs — INTERNAL
- Task execution logs — CONFIDENTIAL
- Never expose credentials or PII found in log output

## Prohibited Actions
- Do NOT disable all log types on production environments
- Do NOT set log retention to indefinite without cost approval
- Do NOT grant broad CloudWatch Logs access to non-operations personnel
- Do NOT delete log groups during active incident investigation
- Do NOT log sensitive data (credentials, PII) in DAG print statements
- Do NOT ignore CloudWatch Logs delivery errors

## Phase 3 — Rollback
- Restore execution role CloudWatch Logs permissions
- Re-enable disabled log types through environment configuration update
- Restore deleted log groups (note: historical data cannot be recovered)
- Revert log level changes to previous configuration
- Verify all five MWAA log types are delivering to CloudWatch
- Document logging configuration changes and their rationale

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
