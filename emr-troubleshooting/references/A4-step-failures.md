---
title: "A4 — Step Failures"
description: "Diagnose EMR step execution failures"
status: active
severity: HIGH
triggers:
  - "step failed"
  - "step execution error"
  - "FAILED step"
  - "job submission failed"
owner: devops-agent
objective: "Identify and resolve EMR step execution failures"
context: "Steps are units of work submitted to the cluster (Spark, Hive, custom JAR, etc.). Failures can stem from application errors, resource issues, configuration problems, or input data issues. Step logs are in S3 at <log-bucket>/<cluster-id>/steps/<step-id>/."
---

## Phase 1 — Triage

MUST:
- Check step status: `aws emr describe-step --cluster-id <id> --step-id <step-id>`
- List all steps: `aws emr list-steps --cluster-id <id>`
- Download step logs from S3: `aws s3 ls s3://<log-bucket>/<cluster-id>/steps/<step-id>/`
- Check step stderr for error messages

SHOULD:
- Check YARN application logs for the step
- Verify input data exists and is accessible
- Check cluster resource availability (memory, CPU)
- Review Spark UI or YARN ResourceManager for application details

MAY:
- Check for concurrent step resource contention
- Review step configuration parameters

## Phase 2 — Remediate

MUST:
- Fix the application error identified in step logs
- Ensure input data is accessible
- Verify resource allocation is sufficient

SHOULD:
- Set action-on-failure to CONTINUE for non-critical steps
- Configure step concurrency appropriately
- Add retry logic for transient failures

MAY:
- Implement step monitoring and alerting
- Create step execution templates

## Common Issues

- symptoms: "Step fails with OOM error"
  diagnosis: "Insufficient memory allocated to executors or driver."
  resolution: "Increase spark.executor.memory or spark.driver.memory. Check YARN limits."

- symptoms: "Step fails with input path not found"
  diagnosis: "Input S3 path doesn't exist or permissions issue."
  resolution: "Verify input path exists. Check instance profile S3 permissions."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix application error and resubmit step | GREEN | Resubmission — non-destructive |
| Ensure input data is accessible | GREEN | Permission verification — read-only |
| Set action-on-failure to CONTINUE | YELLOW | Changes failure behavior — may mask errors |
| Configure step concurrency | YELLOW | Affects resource allocation for running steps |
| Increase executor/driver memory | GREEN | Configuration tuning — applied on next run |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- Step failure affects a time-sensitive data pipeline or SLA
- Step processes sensitive data that may have been partially written

## Data Sensitivity

- **Classification: HIGH**
- Step logs may contain query details, data samples, and error context with sensitive data
- Input/output S3 paths reveal data pipeline architecture
- Spark UI exposes job execution details and data flow patterns
- YARN application logs may contain data processing details

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest deleting step output data without confirming downstream dependencies
- **NEVER** suggest running steps with elevated privileges to bypass permission errors

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Resubmitted step with new configuration | Cancel the running step if configuration is incorrect |
| Changed action-on-failure setting | Revert to previous failure handling behavior |
| Modified step concurrency | Revert concurrency setting to previous value |

## Output Format

```yaml
root_cause: "step_failure — <specific_cause>"
evidence:
  - type: step_status
    content: "<step execution status and error>"
  - type: step_logs
    content: "<relevant log entries>"
severity: HIGH
mitigation:
  immediate: "Fix the application error and resubmit step"
  long_term: "Implement step monitoring and automated retry"
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
  - "NEVER suggest disabling encryption"
  - "NEVER suggest public subnet placement for clusters"
  - "NEVER suggest removing Kerberos authentication"
