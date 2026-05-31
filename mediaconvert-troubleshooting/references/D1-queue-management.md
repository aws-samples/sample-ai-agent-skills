---
title: "D1 — Queue Management Issues"
description: "Diagnose queue configuration and capacity problems"
status: active
severity: MEDIUM
triggers:
  - "queue"
  - "queue full"
  - "job queued"
  - "queue paused"
owner: devops-agent
objective: "Resolve queue configuration, capacity, and management issues"
context: "MediaConvert has on-demand queues (auto-scaling, pay-per-use) and reserved queues (fixed RTS, committed pricing). The default queue is on-demand. Queue status can be ACTIVE or PAUSED."
---

## Phase 1 — Triage

MUST:
- List queues: `aws mediaconvert list-queues --endpoint-url <endpoint>`
- Check queue details: `aws mediaconvert get-queue --name <queue-name> --endpoint-url <endpoint>`
- Check queue status (ACTIVE vs PAUSED)
- List jobs in queue: `aws mediaconvert list-jobs --queue <queue-arn> --status SUBMITTED --endpoint-url <endpoint>`

SHOULD:
- Check CloudWatch queue metrics
- Verify queue pricing plan (on-demand vs reserved)
- Review job priority distribution in the queue

MAY:
- Check for queue-level throttling
- Review historical queue utilization

## Phase 2 — Remediate

MUST:
- If queue paused: resume it: `aws mediaconvert update-queue --name <queue-name> --status ACTIVE --endpoint-url <endpoint>`
- If queue congested: use job priority or submit to a different queue
- If reserved queue full: wait for slots or use on-demand queue for overflow

SHOULD:
- Use separate queues for different workload priorities
- Monitor queue depth with CloudWatch alarms
- Implement job priority for critical content

MAY:
- Set up auto-routing between reserved and on-demand queues
- Implement queue-based cost allocation

## Common Issues

- symptoms: "Jobs stuck in SUBMITTED state"
  diagnosis: "Queue is paused or reserved queue RTS fully occupied."
  resolution: "Check queue status. Resume if paused. Wait for RTS or use on-demand."

- symptoms: "Cannot create additional queues"
  diagnosis: "Queue limit reached (default 10 queues per account)."
  resolution: "Request quota increase or consolidate queues."

## Output Format

```yaml
root_cause: "queue — <specific_cause>"
evidence:
  - type: queue_status
    content: "<queue configuration and state>"
  - type: queue_jobs
    content: "<jobs in queue>"
severity: MEDIUM
mitigation:
  immediate: "Fix queue status or redirect jobs"
  long_term: "Implement queue monitoring and priority management"
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
