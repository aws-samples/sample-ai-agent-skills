---
title: "C1 — Batch Job Failures"
description: "Diagnose batch job execution failures in M2 environments"
status: active
severity: HIGH
triggers: ["batch job failed", "JCL error", "job abend", "condition code error", "batch execution stuck"]
owner: devops-agent
objective: "Resolve batch job execution failures in Mainframe Modernization environments"
context: "Batch jobs in M2 map to JCL job streams. They follow mainframe execution semantics with step-level processing, condition codes, and restart capabilities."
---

## Phase 1 — Triage

MUST:
- List batch job executions: `aws m2 list-batch-job-executions --application-id <app-id> --query 'batchJobExecutions[*].{Id:executionId,JobName:jobName,Status:status,StartTime:startTime}'`
- Get job execution details: `aws m2 get-batch-job-execution --application-id <app-id> --execution-id <exec-id>`
- Review job logs: `aws logs filter-log-events --log-group-name /aws/m2/<environment-name> --filter-pattern "JOB" --start-time <epoch-ms>`

SHOULD:
- Check condition codes for each job step
- Verify input datasets are available and accessible
- Check for resource contention (file locks, dataset allocation)

MAY:
- Compare job execution with mainframe JES output
- Review JCL for syntax or logic errors

## Phase 2 — Remediate

MUST:
- Fix the failing job step based on condition code and error message
- Ensure input datasets are available before job submission
- Verify file permissions and dataset allocation

SHOULD:
- Implement job restart from the failing step (not from the beginning)
- Set up job scheduling with dependency management

MAY:
- Implement automated job monitoring and alerting
- Create job execution dashboards

## Common Issues

- symptoms: "Batch job fails with condition code 12 at step 3"
  diagnosis: "Step 3 encountered a data error or missing input file."
  resolution: "Check step 3 logs for specific error. Verify input datasets. Fix data issue and restart from step 3."

## Safety Ratings

```
safety_ratings:
  - "list-batch-job-executions: GREEN — read-only job listing"
  - "get-batch-job-execution: GREEN — read-only job details"
  - "start-batch-job: YELLOW — initiates batch processing, may modify data"
  - "cancel-batch-job-execution: YELLOW — stops running job, may leave data in inconsistent state"
```

## Escalation Conditions

- Batch job failure affects downstream processing or reporting deadlines
- Job produces incorrect output data affecting business operations
- Job execution time exceeds SLA thresholds
- Recurring failures with no identifiable root cause

## Data Sensitivity

- Batch job execution details: LOW — returns job metadata and status
- Job logs: MEDIUM — may contain business data references and error details
- Input/output datasets: HIGH — contain business data
- JCL definitions: MEDIUM — reveal job processing logic

## Prohibited Actions

- NEVER restart a failed batch job without understanding why it failed
- NEVER cancel a running batch job without assessing data consistency impact
- NEVER modify JCL or job definitions in production without testing
- NEVER assume mainframe condition codes map to Unix exit codes

## Phase 3 — Rollback

1. If a batch job produced incorrect output, identify affected datasets
2. Restore datasets from backups or previous generation (GDG)
3. Fix the root cause before re-running the job
4. Verify output data integrity after successful re-run
5. Document the failure and notify downstream consumers

## Output Format

```yaml
root_cause: "batch_job_failure — <specific_cause>"
evidence:
  - type: job_execution
    content: "<job status, condition codes, and error details>"
severity: HIGH
mitigation:
  immediate: "Fix failing job step"
  long_term: "Implement job monitoring and automated restart"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
