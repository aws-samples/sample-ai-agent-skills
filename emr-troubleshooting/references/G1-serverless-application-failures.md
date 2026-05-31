---
title: "G1 — EMR Serverless Application Failures"
description: "Diagnose EMR Serverless application and job failures"
status: active
severity: HIGH
triggers:
  - "serverless application error"
  - "EMR Serverless job failed"
  - "serverless application creation"
  - "serverless job submission"
owner: devops-agent
objective: "Identify and resolve EMR Serverless application failures"
context: "EMR Serverless applications define resource limits and runtime configuration. Job runs execute within applications. Failures stem from IAM permissions, resource limits, S3 access, application configuration, or runtime errors."
---

## Phase 1 — Triage

MUST:
- Check application status: `aws emr-serverless get-application --application-id <id>`
- Check job run status: `aws emr-serverless get-job-run --application-id <app-id> --job-run-id <job-id>`
- Verify execution role permissions
- Check job run logs in S3 or CloudWatch

SHOULD:
- Verify application resource limits (max CPU, memory)
- Check for pre-initialized capacity configuration
- Verify S3 access for input/output
- Review Spark/Hive configuration overrides

MAY:
- Check EMR Serverless service quotas
- Review application network configuration

## Phase 2 — Remediate

MUST:
- Fix execution role IAM permissions
- Ensure resource limits are sufficient for the workload
- Fix S3 access issues

SHOULD:
- Configure appropriate max resource limits
- Set up CloudWatch logging for debugging
- Test with a simple job before complex workloads

MAY:
- Configure pre-initialized capacity for reduced cold start
- Implement job monitoring and alerting

## Common Issues

- symptoms: "Job run fails with access denied"
  diagnosis: "Execution role lacks required permissions."
  resolution: "Add S3, Glue, and CloudWatch permissions to the execution role."

- symptoms: "Job run exceeds resource limits"
  diagnosis: "Application max CPU or memory too low."
  resolution: "Increase application resource limits."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix execution role IAM permissions | YELLOW | Changes role permissions — review scope |
| Increase application resource limits | YELLOW | Changes capacity — cost increase |
| Fix S3 access issues | GREEN | Permission fix — non-destructive |
| Configure pre-initialized capacity | YELLOW | Reserves resources — ongoing cost |
| Set up CloudWatch logging | GREEN | Monitoring — non-destructive |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- Serverless application failure affects production data pipeline
- Resource limit changes significantly impact cost

## Data Sensitivity

- **Classification: HIGH**
- Execution role permissions reveal data access boundaries
- Application configuration exposes resource allocation and processing capacity
- Job run logs may contain data processing details and query results
- S3 input/output paths reveal data pipeline architecture

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest deleting EMR Serverless applications with active job runs
- **NEVER** suggest granting overly permissive IAM policies to execution roles

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Modified execution role permissions | Revert IAM policy to previous version |
| Increased application resource limits | Reduce limits to previous values if cost is a concern |
| Configured pre-initialized capacity | Remove pre-initialized capacity to stop incurring charges |

## Output Format

```yaml
root_cause: "serverless_application — <specific_cause>"
evidence:
  - type: application_status
    content: "<application configuration>"
  - type: job_status
    content: "<job run status and error>"
severity: HIGH
mitigation:
  immediate: "Fix permissions or resource limits"
  long_term: "Create application templates and monitoring"
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
