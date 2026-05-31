---
title: "D1 — MWAA Worker Scaling Issues"
description: "Diagnose worker auto-scaling problems in MWAA"
status: active
severity: MEDIUM
triggers:
  - "workers not scaling"
  - "not enough workers"
  - "scaling slow"
  - "max workers"
  - "min workers"
owner: devops-agent
objective: "Identify why MWAA workers are not scaling as expected and optimize scaling configuration"
context: "MWAA auto-scales workers based on task queue depth. Min and max workers are configurable. Scaling takes several minutes as new Fargate tasks launch. Scaling issues manifest as growing task queues or underutilized workers."
---

## Phase 1 — Triage

MUST:
- Check worker configuration: `aws mwaa get-environment --name <env-name> --query 'Environment.{MinWorkers:MinWorkers,MaxWorkers:MaxWorkers,EnvironmentClass:EnvironmentClass}'`
- Check running tasks metric: `aws cloudwatch get-metric-statistics --namespace AWS/MWAA --metric-name RunningTasks --dimensions Name=Environment,Value=<env-name> --start-time <start> --end-time <end> --period 300 --statistics Average,Maximum`
- Check queued tasks metric: `aws cloudwatch get-metric-statistics --namespace AWS/MWAA --metric-name QueuedTasks --dimensions Name=Environment,Value=<env-name> --start-time <start> --end-time <end> --period 300 --statistics Average,Maximum`
- Check scheduler logs for task distribution: `aws logs filter-log-events --log-group-name airflow-<env-name>-Scheduler --filter-pattern "executor" --start-time <epoch-ms>`

SHOULD:
- Check environment class limits (mw1.small, mw1.medium, mw1.large)
- Verify no tasks are stuck in queued state indefinitely
- Check worker logs for resource exhaustion: `aws logs filter-log-events --log-group-name airflow-<env-name>-Worker --filter-pattern "resource" --start-time <epoch-ms>`

MAY:
- Compare task throughput before and after scaling events
- Check if task pools are limiting concurrency

## Phase 2 — Remediate

MUST:
- Increase max_workers if consistently hitting the limit
- Upgrade environment class if workers are resource-constrained
- Verify min_workers is set appropriately for baseline workload

SHOULD:
- Use task pools to manage concurrency for resource-intensive tasks
- Stagger DAG schedules to avoid burst task creation
- Monitor QueuedTasks metric and set CloudWatch alarms

MAY:
- Consider splitting workloads across multiple environments
- Optimize task execution time to reduce worker demand

## Common Issues

- symptoms: "Tasks queued for long periods before executing"
  diagnosis: "Max workers reached and scaling cannot add more capacity."
  resolution: "Increase max_workers or optimize task execution time."

- symptoms: "Workers scale up but tasks still slow"
  diagnosis: "Environment class too small — workers lack CPU/memory."
  resolution: "Upgrade environment class (mw1.small → mw1.medium → mw1.large)."

- symptoms: "Workers not scaling down after peak"
  diagnosis: "Normal behavior — MWAA keeps workers for a cooldown period before scaling down."
  resolution: "This is expected. Workers scale down after the cooldown period."

## Output Format

```yaml
root_cause: "worker_scaling_issue — <specific_cause>"
evidence:
  - type: worker_config
    content: "<min/max workers and environment class>"
  - type: metrics
    content: "<RunningTasks and QueuedTasks metrics>"
  - type: scheduler_logs
    content: "<task distribution details>"
severity: MEDIUM
mitigation:
  immediate: "Adjust worker limits or environment class"
  long_term: "Implement monitoring and auto-alerting for queue depth"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — scaling operations do not expose data
- Service Disruption Risk: MEDIUM — insufficient workers cause task queuing and delays
- Compliance Impact: LOW — scaling issues primarily affect performance SLAs

## Escalation Conditions
- Workers fail to scale up despite increasing task queue
- Worker auto-scaling causes repeated environment instability
- Maximum worker count reached and tasks remain queued
- Workers crash repeatedly during scale-up operations
- Cost anomalies detected due to unexpected worker scaling

## Data Sensitivity
- Worker count and scaling metrics — INTERNAL
- Task queue depth metrics — INTERNAL
- Worker resource utilization — INTERNAL
- Auto-scaling configuration — INTERNAL
- Never expose internal worker IP addresses

## Prohibited Actions
- Do NOT set maximum workers to an excessively high number without cost approval
- Do NOT manually terminate worker containers
- Do NOT modify auto-scaling parameters during active high-load periods
- Do NOT disable auto-scaling on production environments without a scaling plan
- Do NOT ignore worker scaling alerts and metrics
- Do NOT set minimum workers to zero on production environments

## Phase 3 — Rollback
- Revert worker min/max configuration to previous values
- Pause non-critical DAGs to reduce worker demand
- Trigger environment update to reset worker fleet if workers are unhealthy
- Restore previous auto-scaling configuration
- Monitor task queue depth and worker count after changes
- Document scaling event and adjust capacity planning accordingly

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
