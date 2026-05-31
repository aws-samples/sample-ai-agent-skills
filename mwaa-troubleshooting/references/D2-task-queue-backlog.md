---
title: "D2 — MWAA Task Queue Backlog"
description: "Diagnose and resolve task queue backlog in MWAA"
status: active
severity: MEDIUM
triggers:
  - "task queue growing"
  - "tasks stuck in queue"
  - "backlog"
  - "tasks not starting"
  - "queued tasks increasing"
owner: devops-agent
objective: "Identify why tasks are accumulating in the queue and restore normal task throughput"
context: "Task queue backlog occurs when task creation rate exceeds worker processing capacity. Causes include insufficient workers, resource-intensive tasks, scheduler bottlenecks, or task pool limits."
---

## Phase 1 — Triage

MUST:
- Check queued tasks: `aws cloudwatch get-metric-statistics --namespace AWS/MWAA --metric-name QueuedTasks --dimensions Name=Environment,Value=<env-name> --start-time <start> --end-time <end> --period 60 --statistics Average,Maximum`
- Check running tasks: `aws cloudwatch get-metric-statistics --namespace AWS/MWAA --metric-name RunningTasks --dimensions Name=Environment,Value=<env-name> --start-time <start> --end-time <end> --period 60 --statistics Average,Maximum`
- Check worker config: `aws mwaa get-environment --name <env-name> --query 'Environment.{MinWorkers:MinWorkers,MaxWorkers:MaxWorkers}'`
- Check scheduler logs for bottlenecks: `aws logs filter-log-events --log-group-name airflow-<env-name>-Scheduler --filter-pattern "pool" --start-time <epoch-ms>`

SHOULD:
- Check if specific DAGs are flooding the queue with many tasks
- Verify task pool configurations via CLI token: `pools list`
- Check parallelism and dag_concurrency settings in environment configuration overrides

MAY:
- Analyze task duration distribution to identify long-running tasks
- Check if external dependencies (APIs, databases) are causing task slowdowns

## Phase 2 — Remediate

MUST:
- Increase max_workers if queue depth consistently exceeds capacity
- Clear stuck tasks if they are blocking the queue: use CLI token with `tasks clear <dag_id>`
- Fix any tasks that are hanging indefinitely

SHOULD:
- Configure task pools to limit concurrency for resource-heavy tasks
- Stagger DAG schedules to smooth task creation rate
- Set appropriate task timeouts to prevent indefinite hangs

MAY:
- Split large DAGs into smaller, more manageable DAGs
- Use priority_weight to ensure critical tasks run first

## Common Issues

- symptoms: "QueuedTasks metric steadily increasing"
  diagnosis: "Task creation rate exceeds worker processing capacity."
  resolution: "Increase max_workers and/or optimize task execution time."

- symptoms: "Tasks queued but workers appear idle"
  diagnosis: "Task pool limit reached or scheduler not distributing tasks."
  resolution: "Check task pool slots and increase if needed. Verify scheduler health."

- symptoms: "Sudden spike in queued tasks"
  diagnosis: "Multiple DAGs with catchup=True triggered simultaneously."
  resolution: "Set catchup=False or stagger DAG start_dates."

## Output Format

```yaml
root_cause: "task_queue_backlog — <specific_cause>"
evidence:
  - type: queue_metrics
    content: "<QueuedTasks and RunningTasks over time>"
  - type: worker_config
    content: "<min/max workers>"
  - type: task_pools
    content: "<pool configurations and usage>"
severity: MEDIUM
mitigation:
  immediate: "Scale workers and clear stuck tasks"
  long_term: "Optimize task scheduling and implement queue depth alerting"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — queue backlog is an operational concern
- Service Disruption Risk: HIGH — large backlogs can cascade into missed SLAs
- Compliance Impact: MEDIUM — backlogged tasks may miss data processing deadlines

## Escalation Conditions
- Task queue backlog exceeds 100 tasks for more than 30 minutes
- Backlog causes critical DAG runs to miss their SLA
- Queue backlog caused by infrastructure issue rather than workload spike
- Workers are available but tasks remain queued (scheduler issue)
- Backlog recovery time exceeds acceptable business thresholds

## Data Sensitivity
- Queue metrics and task counts — INTERNAL
- Task priority and pool configurations — INTERNAL
- SLA miss notifications — INTERNAL
- Worker utilization data — INTERNAL
- Never expose task payload data from queue inspection

## Prohibited Actions
- Do NOT clear the entire task queue to resolve backlog
- Do NOT kill running tasks to free up worker slots without approval
- Do NOT disable task concurrency limits without understanding impact
- Do NOT ignore persistent queue backlogs on production environments
- Do NOT manually reorder task queue priority without DAG owner approval
- Do NOT increase parallelism beyond what the environment can support

## Phase 3 — Rollback
- Restore previous concurrency and parallelism settings
- Re-queue tasks that were incorrectly cleared or failed during backlog
- Revert pool size changes if they caused resource contention
- Pause low-priority DAGs until backlog is cleared
- Monitor queue depth and task completion rate after adjustments
- Document backlog event timeline and root cause for capacity planning

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
