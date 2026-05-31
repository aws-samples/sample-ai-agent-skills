---
title: "B2 — MWAA Task Failures"
description: "Diagnose Airflow task execution failures in MWAA"
status: active
severity: HIGH
triggers:
  - "task failed"
  - "task error"
  - "task retry"
  - "task stuck"
  - "upstream_failed"
owner: devops-agent
objective: "Identify why Airflow tasks fail during execution and resolve the root cause"
context: "Task failures in MWAA can stem from code errors, resource limits, IAM permission issues, network connectivity problems, or worker resource exhaustion. Task logs are in the Task CloudWatch log group."
---

## Phase 1 — Triage

MUST:
- Check task logs: `aws logs filter-log-events --log-group-name airflow-<env-name>-Task --filter-pattern "<task_id>" --start-time <epoch-ms>`
- Check task state via CLI token: use `aws mwaa create-cli-token --name <env-name>` then `tasks states-for-dag-run <dag_id> <execution_date>`
- Check worker logs for errors: `aws logs filter-log-events --log-group-name airflow-<env-name>-Worker --filter-pattern "ERROR" --start-time <epoch-ms>`
- Verify execution role has permissions for task operations: `aws iam list-attached-role-policies --role-name <execution-role-name>`

SHOULD:
- Check if task is timing out: review task duration vs timeout setting
- Check worker resource metrics: `aws cloudwatch get-metric-statistics --namespace AWS/MWAA --metric-name RunningTasks --dimensions Name=Environment,Value=<env-name> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check for upstream task failures that cascade

MAY:
- Test task locally with same dependencies
- Check if task failure correlates with environment update or scaling event

## Phase 2 — Remediate

MUST:
- Fix code errors identified in task logs
- Grant missing IAM permissions to the execution role
- Fix network connectivity issues for tasks calling external services

SHOULD:
- Increase task timeout if task legitimately needs more time
- Add retry logic with exponential backoff for transient failures
- Monitor worker resource utilization and scale if needed

MAY:
- Split long-running tasks into smaller subtasks
- Use task pools to limit concurrency for resource-intensive tasks

## Common Issues

- symptoms: "Task fails with AccessDeniedException"
  diagnosis: "Execution role lacks permissions for the AWS service the task is calling."
  resolution: "Add required IAM permissions to the MWAA execution role."

- symptoms: "Task fails with connection timeout"
  diagnosis: "Worker cannot reach the target service due to VPC/security group configuration."
  resolution: "Verify security group outbound rules and NAT gateway connectivity."

- symptoms: "Task killed with SIGTERM"
  diagnosis: "Task exceeded memory or time limits on the worker."
  resolution: "Optimize task resource usage or increase environment class for more worker resources."

## Output Format

```yaml
root_cause: "task_failure — <specific_cause>"
evidence:
  - type: task_logs
    content: "<error messages from task execution>"
  - type: worker_logs
    content: "<worker-level errors>"
  - type: iam_permissions
    content: "<missing permissions>"
severity: HIGH
mitigation:
  immediate: "Fix the identified code, permission, or connectivity issue"
  long_term: "Add monitoring and alerting for task failure patterns"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — task logs may contain sensitive data from upstream systems
- Service Disruption Risk: MEDIUM — failed tasks can block downstream workflows
- Compliance Impact: MEDIUM — failed data pipeline tasks may affect data freshness SLAs

## Escalation Conditions
- Tasks fail with OOM (Out of Memory) errors repeatedly
- Task failures correlate with MWAA infrastructure issues (worker crashes)
- Critical path tasks fail affecting business-critical data pipelines
- Task failure rate exceeds normal baseline by more than 50%
- Tasks fail with permission errors after IAM policy changes

## Data Sensitivity
- Task execution logs — INTERNAL (may contain CONFIDENTIAL data)
- XCom values passed between tasks — CONFIDENTIAL
- Connection credentials used by tasks — RESTRICTED
- Task parameters and configurations — INTERNAL
- Never expose database connection strings from task logs

## Prohibited Actions
- Do NOT retry failed tasks without understanding the root cause
- Do NOT increase worker resources without analyzing memory/CPU patterns
- Do NOT disable task-level retries on production-critical DAGs
- Do NOT clear task state on running tasks
- Do NOT expose task logs containing credentials to unauthorized users
- Do NOT modify task parameters in production without change approval

## Phase 3 — Rollback
- Mark failed tasks as `failed` and re-trigger from the last successful task
- Restore previous DAG version if task failure was caused by code change
- Revert connection or variable changes that may have caused task failures
- Scale down worker count if over-provisioning caused resource contention
- Clear stuck task instances using the Airflow CLI or UI
- Document task failure patterns and update alerting thresholds

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
