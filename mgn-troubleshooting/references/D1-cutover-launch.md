---
title: "D1 — Cutover Launch"
description: "Diagnose cutover launch failures"
status: active
severity: CRITICAL
triggers: ["cutover failed", "start cutover error", "cutover launch"]
owner: devops-agent
objective: "Resolve cutover launch failures"
context: "Cutover creates the final production instance from replicated data. Failures block migration completion."
---

## Phase 1 — Triage

MUST:
- Check job status: `aws mgn describe-jobs --filters 'fromDate=2024-01-01T00:00:00Z'`
- Get job logs: `aws mgn describe-job-log-items --job-id <job-id>`
- Verify source server state is READY_FOR_CUTOVER
- Check launch configuration: `aws mgn get-launch-configuration --source-server-id <id>`

SHOULD:
- Verify EC2 capacity in target AZ
- Check IAM service role permissions
- Verify replication is in CONTINUOUS state (not lagging)

MAY:
- Review CloudTrail for RunInstances errors

## Phase 2 — Remediate

MUST:
- Fix issues identified in job logs
- Ensure server is in READY_FOR_CUTOVER state
- Verify launch config is valid

SHOULD:
- Run test launch first to validate
- Retry after fixing configuration

MAY:
- Try alternative AZ if capacity issue

## Common Issues

- symptoms: "Cutover fails — server in READY_FOR_TEST state"
  diagnosis: "Server must be marked READY_FOR_CUTOVER before cutover."
  resolution: "Mark as ready for cutover: aws mgn change-server-life-cycle-state --source-server-id <id> --life-cycle 'state=READY_FOR_CUTOVER'"

## Output Format

```yaml
root_cause: "cutover_launch — <specific_cause>"
evidence:
  - type: job_logs
    content: "<cutover job error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix blocker and retry cutover"
  long_term: "Always run test launch before cutover"
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

- **NEVER** finalize a cutover without confirmed application validation and customer sign-off
- **NEVER** disconnect replication on source servers before cutover completion is verified
- **NEVER** delete source server records that have active replication or pending cutover operations
- **NEVER** modify launch templates for production migrations without testing in a staging environment first
- **NEVER** skip post-migration validation steps even if the launch appears successful

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
