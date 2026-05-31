---
title: "A1 — Job Failures"
description: "Diagnose why AWS Batch jobs are failing"
status: active
severity: HIGH
triggers:
  - "job failed"
  - "exit code non-zero"
  - "container error"
  - "job FAILED status"
  - "job execution error"
owner: devops-agent
objective: "Identify why jobs are failing and resolve the issue"
context: "Job failures occur due to: application errors (exit code 1), OOM killed (exit code 137), container image issues, missing environment variables, IAM permission errors, or resource constraints."
---

## Phase 1 — Triage

MUST:
- Check job details: `aws batch describe-jobs --jobs <job-id> --query 'jobs[*].{Status:status,Reason:statusReason,ExitCode:container.exitCode,ContainerReason:container.reason,LogStream:container.logStreamName}'`
- Check container logs: `aws logs get-log-events --log-group-name /aws/batch/job --log-stream-name <log-stream>`
- Interpret exit code: 0=success, 1=app error, 137=OOM, 139=segfault, 143=SIGTERM
- Check job definition: `aws batch describe-job-definitions --job-definitions <job-def-name:revision>`

SHOULD:
- Check if container image exists and is pullable
- Verify environment variables are set correctly
- Check resource allocation (vCPU, memory) is sufficient
- Review job retry attempts if retry strategy configured

MAY:
- Check for transient infrastructure errors
- Review job submission parameters

## Phase 2 — Remediate

MUST:
- If exit code 137: increase memory in job definition
- If exit code 1: fix application code (check logs for specific error)
- If image pull error: verify ECR/Docker Hub access
- If permission error: fix IAM execution role

SHOULD:
- Update job definition with corrected settings: `aws batch register-job-definition --job-definition-name <name> --type container --container-properties '{"image":"<image>","vcpus":2,"memory":4096}'`
- Configure retry strategy for transient failures

MAY:
- Implement health checks in container
- Set up CloudWatch alarms on job failures

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Increase memory in job definition | GREEN | New revision — non-destructive |
| Fix application code | GREEN | Code fix — applied on next run |
| Verify ECR/Docker Hub access | GREEN | Verification — read-only |
| Fix IAM execution role | YELLOW | Permission change — review scope |
| Configure retry strategy | GREEN | Adds resilience — non-destructive |

## Escalation Conditions

- Compute environment runs production batch jobs
- Job failure affects downstream data pipelines or SLA
- Exit code 137 (OOM) indicates persistent resource sizing issue
- Image pull failures block all job executions

## Data Sensitivity

- **Classification: MEDIUM**
- Container logs may contain application output with sensitive data
- Job definitions reveal container images, commands, and environment variables
- Exit codes and failure reasons expose application behavior
- Environment variables may reference secrets or configuration endpoints

## Prohibited Actions

- **NEVER** suggest deleting a compute environment with running jobs
- **NEVER** suggest deregistering job definitions that are actively referenced by running jobs
- **NEVER** suggest embedding secrets directly in job definition environment variables
- **NEVER** suggest running containers in privileged mode to resolve permission errors

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Registered new job definition revision | Submit jobs using previous revision number |
| Modified IAM execution role | Revert role policy to previous version |
| Configured retry strategy | Remove retry strategy from job definition if not needed |

## Output Format

```yaml
root_cause: "job_failure — <specific_cause>"
evidence:
  - type: exit_code
    content: "<exit code and reason>"
  - type: container_logs
    content: "<relevant log output>"
severity: HIGH
mitigation:
  immediate: "Fix the specific failure cause"
  long_term: "Implement retry strategies and monitoring"
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
  - "NEVER suggest overly broad compute environment permissions"
  - "NEVER suggest disabling job queue scheduling"
  - "NEVER suggest public subnet placement for compute environments"
