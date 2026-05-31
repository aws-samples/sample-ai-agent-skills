---
title: "B3 — MWAA DAG Scheduling Issues"
description: "Diagnose DAG scheduling problems in MWAA"
status: active
severity: MEDIUM
triggers:
  - "DAG not running"
  - "schedule not triggering"
  - "missed schedule"
  - "DAG paused"
  - "next run not scheduled"
owner: devops-agent
objective: "Identify why DAGs are not being scheduled as expected and fix the scheduling configuration"
context: "DAG scheduling issues can stem from paused DAGs, incorrect schedule_interval, timezone mismatches, scheduler lag, or catchup behavior. The scheduler processes DAGs periodically and creates DAG runs based on the schedule."
---

## Phase 1 — Triage

MUST:
- Check if DAG is paused: use CLI token with `dags list` and check `is_paused` status
- Check DAG schedule_interval in the DAG file: `aws s3 cp s3://<bucket>/<dags-path>/<dag-file> - | grep schedule`
- Check scheduler logs: `aws logs filter-log-events --log-group-name airflow-<env-name>-Scheduler --filter-pattern "<dag_id>" --start-time <epoch-ms>`
- Verify DAG has a valid start_date: check DAG file for `start_date` parameter
- Check next DAG run: use CLI token with `dags next-execution <dag_id>`

SHOULD:
- Check if catchup is enabled and causing backfill: look for `catchup=True` in DAG definition
- Verify timezone configuration: check `AIRFLOW__CORE__DEFAULT_TIMEZONE` environment variable
- Check scheduler heartbeat: `aws logs filter-log-events --log-group-name airflow-<env-name>-Scheduler --filter-pattern "heartbeat" --start-time <epoch-ms>`

MAY:
- Check if max_active_runs is limiting new DAG runs
- Verify DAG file modification time in S3: `aws s3api head-object --bucket <bucket> --key <dags-path>/<dag-file>`

## Phase 2 — Remediate

MUST:
- Unpause the DAG if it was paused
- Fix schedule_interval syntax (cron or timedelta)
- Set a valid start_date that is not in the future

SHOULD:
- Set `catchup=False` if backfill is not desired
- Configure timezone explicitly if UTC is not intended
- Verify max_active_runs allows new runs to be created

MAY:
- Manually trigger a DAG run to verify execution works: use CLI token with `dags trigger <dag_id>`
- Adjust scheduler parsing interval if DAG pickup is slow

## Common Issues

- symptoms: "DAG exists but no runs are being created"
  diagnosis: "DAG is paused or start_date is in the future."
  resolution: "Unpause the DAG and ensure start_date is in the past."

- symptoms: "DAG runs at unexpected times"
  diagnosis: "Timezone mismatch between DAG definition and expected schedule."
  resolution: "Set timezone explicitly in DAG or configure AIRFLOW__CORE__DEFAULT_TIMEZONE."

- symptoms: "Hundreds of DAG runs created at once"
  diagnosis: "catchup=True with a start_date far in the past."
  resolution: "Set catchup=False or adjust start_date to a recent date."

## Output Format

```yaml
root_cause: "scheduling_issue — <specific_cause>"
evidence:
  - type: dag_config
    content: "<schedule_interval, start_date, catchup settings>"
  - type: scheduler_logs
    content: "<scheduler processing details>"
  - type: dag_state
    content: "<paused status and next execution>"
severity: MEDIUM
mitigation:
  immediate: "Fix the scheduling configuration and unpause if needed"
  long_term: "Standardize DAG scheduling patterns and timezone handling"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — scheduling issues do not directly expose data
- Service Disruption Risk: HIGH — scheduling failures can halt all workflow execution
- Compliance Impact: MEDIUM — missed schedules may violate data processing SLAs

## Escalation Conditions
- Scheduler is completely unresponsive for more than 15 minutes
- DAG runs are not being created despite active schedules
- Scheduling lag exceeds acceptable thresholds for critical workflows
- Scheduler heartbeat missing in CloudWatch metrics
- Multiple DAGs show unexpected scheduling behavior simultaneously

## Data Sensitivity
- Scheduler logs — INTERNAL
- DAG schedule configurations — INTERNAL
- Task queue metrics — INTERNAL
- Airflow metadata database state — CONFIDENTIAL
- Never expose scheduler internal state to unauthorized users

## Prohibited Actions
- Do NOT manually manipulate the Airflow metadata database to fix scheduling
- Do NOT disable the scheduler to troubleshoot without a maintenance window
- Do NOT set overly aggressive schedule intervals that overwhelm the scheduler
- Do NOT ignore scheduler health metrics and alerts
- Do NOT run backfill operations during peak scheduling periods
- Do NOT modify scheduler configuration without understanding impact on running DAGs

## Phase 3 — Rollback
- If scheduler is stuck, trigger an environment update to restart MWAA components
- Pause non-critical DAGs to reduce scheduler load
- Revert any recent Airflow configuration overrides affecting the scheduler
- Clear orphaned DAG runs that may be blocking the scheduler
- Restore scheduler-related configuration to last known working values
- Monitor scheduler heartbeat and DAG processing time after recovery

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
