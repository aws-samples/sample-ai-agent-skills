---
title: "C2 — Hybrid Job Failures"
description: "Diagnose Amazon Braket hybrid job execution failures"
status: active
severity: HIGH
triggers: ["hybrid job failed", "braket job error", "job FAILED status", "algorithm script error"]
owner: devops-agent
objective: "Resolve hybrid job failures spanning classical and quantum components"
context: "Hybrid jobs combine classical compute (EC2 via SageMaker) with quantum device access. Failures can originate from either component."
---

## Phase 1 — Triage

MUST:
- Get job details: `aws braket get-job --job-arn <arn>`
- Check job failureReason field
- Review job logs: `aws logs filter-log-events --log-group-name /aws/braket/jobs/<job-name> --limit 50`

SHOULD:
- Verify IAM role has permissions for both SageMaker and Braket
- Check if the algorithm script has syntax or import errors
- Verify the specified instance type is available in the region

MAY:
- Check SageMaker service quotas for the instance type
- Review S3 input/output bucket configurations

## Phase 2 — Remediate

MUST:
- Fix algorithm script errors identified in logs
- Ensure IAM role has braket:*, sagemaker:*, s3:*, and logs:* permissions
- Verify container image or framework version compatibility

SHOULD:
- Test algorithm script locally before submitting as a job
- Use Braket SDK AwsQuantumJob.create() with correct parameters
- Verify quantum device is available during expected job runtime

MAY:
- Implement checkpointing in algorithm script for long-running jobs
- Set up CloudWatch alarms for job failures

## Safety Ratings

```
safety_ratings:
  - "get-job: GREEN — read-only job status"
  - "filter-log-events: GREEN — read-only log retrieval"
  - "create-job (resubmit): YELLOW — incurs EC2 instance and quantum device charges"
  - "cancel-job: YELLOW — stops running job, partial charges apply"
```

## Escalation Conditions

- Job fails with internal service errors not related to user code
- IAM permissions are correct but job cannot access quantum devices
- Job runs successfully on local simulator but fails in cloud
- Persistent container image pull failures

## Data Sensitivity

- `aws braket get-job`: LOW — returns job metadata and status
- `aws logs filter-log-events`: MEDIUM — may contain algorithm parameters and intermediate results
- S3 job artifacts: MEDIUM — may contain model parameters and training data

## Prohibited Actions

- NEVER resubmit failed jobs without fixing the root cause — each submission incurs compute charges
- NEVER modify the IAM role to use AdministratorAccess as a troubleshooting shortcut
- NEVER delete job logs before investigating the failure
- NEVER assume classical and quantum failures have the same root cause

## Phase 3 — Rollback

1. Cancel running job if it is stuck: `aws braket cancel-job --job-arn <arn>`
2. If IAM role was modified, revert to the original policy
3. If algorithm script was changed, restore from version control
4. Clean up any partial output artifacts in S3
5. Document the failure mode and resolution for the team

## Common Issues

- symptoms: "Job fails immediately with 'AlgorithmError' in logs"
  diagnosis: "Algorithm script has a Python error (import, syntax, or runtime)."
  resolution: "Review CloudWatch logs for the Python traceback. Fix the script and resubmit."

- symptoms: "Job fails with 'ResourceLimitExceeded' error"
  diagnosis: "SageMaker instance type quota exceeded in the region."
  resolution: "Request a quota increase for the instance type or use a different instance type."

## Output Format

```yaml
root_cause: "hybrid_job_failure — <specific_cause>"
evidence:
  - type: job_error
    content: "<failureReason and log excerpts>"
severity: HIGH
mitigation:
  immediate: "Fix algorithm script or configuration"
  long_term: "Implement local testing pipeline and checkpointing"
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
