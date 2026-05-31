---
title: "E2 — Job Management Issues"
description: "Diagnose job lifecycle and management problems"
status: active
severity: LOW
triggers:
  - "job stuck"
  - "cannot stop job"
  - "job list error"
owner: devops-agent
objective: "Identify and resolve job management issues"
context: "Comprehend jobs have lifecycle states: SUBMITTED, IN_PROGRESS, COMPLETED, FAILED, STOPPED. Jobs can be stopped but not restarted. Results available after completion."
---

## Phase 1 — Triage

MUST:
- List jobs: `aws comprehend list-entities-detection-jobs --filter '{"JobStatus":"IN_PROGRESS"}'`
- Check specific job: `aws comprehend describe-entities-detection-job --job-id <job-id>`
- Verify job status and timestamps
- Check for stuck jobs (IN_PROGRESS for extended time)

SHOULD:
- Review job output location
- Check for concurrent job limits
- Monitor job duration

MAY:
- Check CloudTrail for job lifecycle events
- Review job cost estimates

## Phase 2 — Remediate

MUST:
- Stop stuck jobs: `aws comprehend stop-entities-detection-job --job-id <job-id>`
- Resubmit failed jobs after fixing root cause
- Store results from completed jobs

SHOULD:
- Implement job monitoring and alerting
- Set up job timeout handling

MAY:
- Implement job orchestration with Step Functions

## Common Issues

- symptoms: "Job stuck in IN_PROGRESS"
  diagnosis: "Large input or service-side processing delay."
  resolution: "Wait or stop and resubmit with smaller input."

- symptoms: "Cannot find job results"
  diagnosis: "Job not completed or output location incorrect."
  resolution: "Check job status. Verify S3 output location."

## Output Format

```yaml
root_cause: "job_management — <specific_cause>"
evidence:
  - type: job_status
    content: "<job lifecycle details>"
severity: LOW
mitigation:
  immediate: "Stop stuck jobs or fix configuration"
  long_term: "Implement job lifecycle monitoring"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Job management operations do not expose document content |
| Blast Radius | LOW — Stopping jobs is non-destructive; results from completed jobs are preserved |
| Rollback Complexity | LOW — Jobs can be resubmitted |
| Compliance Scope | LOW — Job lifecycle management has minimal compliance impact |

## Escalation Conditions

- Escalate to operations if jobs are stuck in IN_PROGRESS for abnormally long periods
- Escalate to management if stuck jobs are blocking time-sensitive processing pipelines
- Escalate to AWS Support if jobs cannot be stopped via the API

## Data Sensitivity

- **Job metadata** (IDs, status, timestamps) is low sensitivity
- **Job output locations** point to S3 paths that may contain sensitive results — protect the paths
- **CloudTrail events** for job operations are standard audit records

## PROHIBITED Actions

- Do NOT stop jobs without understanding the impact on downstream processing
- Do NOT delete job output without confirming it has been consumed or is no longer needed
- Do NOT ignore stuck jobs — they may indicate underlying service issues

## Phase 3 — Rollback

- Stop stuck jobs: `aws comprehend stop-entities-detection-job --job-id <job-id>`
- Resubmit jobs after resolving the root cause
- Verify job output completeness before marking processing as done
- Update job orchestration to handle the failure scenario

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
