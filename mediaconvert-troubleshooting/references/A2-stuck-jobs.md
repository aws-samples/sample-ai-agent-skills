---
title: "A2 — Stuck Jobs"
description: "Diagnose jobs stuck in PROGRESSING or SUBMITTED state"
status: active
severity: HIGH
triggers:
  - "stuck job"
  - "job not progressing"
  - "PROGRESSING"
  - "job taking too long"
owner: devops-agent
objective: "Identify why jobs are stuck and resolve processing delays"
context: "Jobs may appear stuck due to queue congestion, large input files, complex encoding settings, or service issues. SUBMITTED means waiting in queue. PROGRESSING means actively encoding."
---

## Phase 1 — Triage

MUST:
- Check job status and phase: `aws mediaconvert get-job --id <job-id> --endpoint-url <endpoint>`
- Check queue status: `aws mediaconvert get-queue --name <queue-name> --endpoint-url <endpoint>`
- List all jobs in queue: `aws mediaconvert list-jobs --status PROGRESSING --queue <queue-arn> --endpoint-url <endpoint>`
- Check job submission time vs current time

SHOULD:
- Check if reserved queue RTS are all occupied
- Verify input file size (large files take longer)
- Check encoding complexity (resolution, codec, passes)

MAY:
- Check AWS Health Dashboard for MediaConvert service issues
- Review CloudWatch processing time metrics

## Phase 2 — Remediate

MUST:
- If queue full (reserved): wait for current jobs or use on-demand queue
- If job genuinely stuck: cancel and resubmit: `aws mediaconvert cancel-job --id <job-id> --endpoint-url <endpoint>`
- If service issue: check AWS Health Dashboard and wait

SHOULD:
- Use job priority to ensure critical jobs process first
- Monitor queue depth with CloudWatch
- Set up job completion notifications

MAY:
- Split large jobs into smaller segments
- Use accelerated transcoding for faster processing

## Common Issues

- symptoms: "Job in SUBMITTED state for hours"
  diagnosis: "Reserved queue RTS fully occupied by other jobs."
  resolution: "Wait for slots or submit to on-demand queue. Increase RTS if persistent."

- symptoms: "Job in PROGRESSING for much longer than expected"
  diagnosis: "Large input file or complex encoding settings (4K, multi-pass, HDR)."
  resolution: "This may be normal. Check input size and encoding complexity."

## Output Format

```yaml
root_cause: "stuck_job — <specific_cause>"
evidence:
  - type: job_status
    content: "<job state and timing>"
  - type: queue_status
    content: "<queue utilization>"
severity: HIGH
mitigation:
  immediate: "Cancel and resubmit or wait for queue capacity"
  long_term: "Implement queue monitoring and job priority management"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete or modify a production job queue while jobs are actively processing
- **NEVER** change IAM roles on job templates without verifying S3 input/output bucket access
- **NEVER** disable DRM or content protection settings on outputs destined for licensed content
- **NEVER** cancel reserved transcoding capacity without assessing active workload commitments
- **NEVER** modify output presets for compliance-regulated content without re-validation

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
