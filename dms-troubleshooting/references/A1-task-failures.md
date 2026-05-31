---
title: "A1 — Task Failures"
description: "Diagnose why a DMS replication task is failing"
status: active
severity: HIGH
triggers:
  - "task failed"
  - "task error"
  - "replication task stopped"
  - "migration failed"
owner: devops-agent
objective: "Identify why a DMS replication task is failing and resolve the issue"
context: "DMS tasks can fail due to endpoint connectivity issues, insufficient permissions, incompatible data types, source/target errors, replication instance resource exhaustion, or task configuration problems. Check task status, last failure message, and CloudWatch logs."
---

## Phase 1 — Triage

MUST:
- Check task status and last failure message: `aws dms describe-replication-tasks --filters Name=replication-task-id,Values=<task-id> --query 'ReplicationTasks[*].{Status:Status,StopReason:StopReason,LastFailureMessage:LastFailureMessage}'`
- Check replication instance health: `aws dms describe-replication-instances --filters Name=replication-instance-id,Values=<instance-id> --query 'ReplicationInstances[*].{Status:ReplicationInstanceStatus,Class:ReplicationInstanceClass,Storage:AllocatedStorage}'`
- Test source endpoint connectivity: `aws dms test-connection --replication-instance-arn <instance-arn> --endpoint-arn <source-endpoint-arn>`
- Test target endpoint connectivity: `aws dms test-connection --replication-instance-arn <instance-arn> --endpoint-arn <target-endpoint-arn>`

SHOULD:
- Check CloudWatch metrics for the replication instance: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CPUUtilization --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check table statistics for error tables: `aws dms describe-table-statistics --replication-task-arn <task-arn> --query 'TableStatistics[?TableState==`TABLE_ERROR`]'`
- Review CloudTrail for task events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=dms.amazonaws.com --max-results 20`

MAY:
- Check DMS task CloudWatch Logs for detailed error messages
- Review task settings JSON for misconfiguration

## Phase 2 — Remediate

MUST:
- Fix endpoint connectivity issues (security groups, credentials, network)
- Resolve data type incompatibilities identified in error messages
- Ensure replication instance has sufficient resources

SHOULD:
- Restart the task with resume processing: `aws dms start-replication-task --replication-task-arn <task-arn> --start-replication-task-type resume-processing`
- Enable detailed CloudWatch logging for the task

MAY:
- Modify task settings to handle specific error scenarios (ErrorBehavior)
- Scale up replication instance if resource-constrained

## Common Issues

- symptoms: "Task stops with 'Last Error Test connection failed'"
  diagnosis: "Endpoint connectivity lost during migration."
  resolution: "Re-test connections. Check security groups, credentials, and network path."

- symptoms: "Task fails with data type conversion error"
  diagnosis: "Source data type cannot be mapped to target type."
  resolution: "Add transformation rules or modify target schema to accommodate the data type."

- symptoms: "Task stops with 'Storage full' error"
  diagnosis: "Replication instance ran out of local storage."
  resolution: "Increase allocated storage on the replication instance. Monitor FreeStorageSpace."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix endpoint connectivity (SGs, credentials, network) | YELLOW | Changes network/auth config — verify before applying |
| Resolve data type incompatibilities | GREEN | Schema/mapping change — non-destructive |
| Restart task with resume processing | YELLOW | Resumes from last checkpoint — verify data consistency |
| Scale up replication instance | YELLOW | Instance modification — brief downtime during apply |
| Enable detailed CloudWatch logging | GREEN | Monitoring — non-destructive |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- Task failure causes data sync gap between source and target
- Replication instance modification affects other running tasks

## Data Sensitivity

- **Classification: HIGH**
- Task error messages may contain source/target connection details and credentials
- Table statistics reveal database schema and data volumes
- CloudWatch metrics expose replication patterns and data flow rates
- Endpoint configurations contain server addresses, ports, and database names

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest stopping a task without understanding the impact on data consistency
- **NEVER** suggest restarting full load when resume-processing is possible

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Fixed endpoint credentials | Revert credentials if new ones cause issues |
| Restarted task with resume processing | Stop task if resume causes data inconsistency |
| Scaled up replication instance | Scale down after issue is resolved if cost is a concern |
| Modified task settings (ErrorBehavior) | Revert task settings to previous configuration |

## Output Format

```yaml
root_cause: "task_failure — <specific_cause>"
evidence:
  - type: task_status
    content: "<task status and last failure message>"
  - type: endpoint_connectivity
    content: "<connection test results>"
  - type: instance_health
    content: "<replication instance metrics>"
severity: HIGH
mitigation:
  immediate: "Fix the specific failure cause and resume the task"
  long_term: "Enable monitoring and alerting on task status and instance metrics"
```

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
  - "NEVER suggest disabling SSL for replication endpoints"
  - "NEVER suggest public replication instances"
  - "NEVER suggest deleting replication tasks without data verification"
